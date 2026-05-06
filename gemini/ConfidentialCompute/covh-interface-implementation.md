# CoVH 接口功能与实现说明

本文总结 Salus 项目中 CoVH，也就是 COVE Host 接口，在 Rust 代码里的实现方法和实现细节。这里的 CoVH 指 Host/Tellus 用来管理 TVM/GuestVM 的 SBI host 侧接口。

当前工作树中的 `sbi-rs` 子模块没有展开，因此本文以 Salus 实际实现入口 `src/vm.rs::handle_cove_host_msg()` 中匹配到的接口为准。

## 总览

CoVH 的调用方是 Host VM，在测试 workload 中就是 Tellus。Tellus 通过这些接口请求 Salus：

- 查询 TSM/Salus 能力。
- 将 Host 自己的页面转换成可捐赠页面。
- 创建、装载、finalize、运行、销毁 GuestVM。
- 给 GuestVM 添加页表页、vCPU、measured pages、zero pages、shared pages。
- 做运行期页表维护：block、fence、remove、promote、demote。

整体关系：

```text
Tellus / Host VM
  |
  | CoVH SBI calls
  v
Salus Vm::handle_cove_host_msg()
  |
  +--> Guest tracking: GuestVm / Guests
  +--> Page ownership: PageTracker / PageState
  +--> Guest address space: VmPages / GuestStagePageTable
  +--> vCPU execution: VmCpu / ActiveVmCpu
```

CoVH 在 Salus 中的分发入口：

```rust
fn handle_cove_host_msg(
    &self,
    host_func: CoveHostFunction,
    active_vcpu: &mut ActiveVmCpu<T>,
) -> EcallAction
```

它根据 `CoveHostFunction` 调用具体实现函数，并把结果转换成 `EcallAction`。

## 重要内部结构

### `Vm<T>`

`src/vm.rs` 中：

```rust
pub struct Vm<T: GuestStagePagingMode> {
    vcpus: VmCpus,
    vm_pages: VmPages<T>,
    guests: Option<Guests<T>>,
    attestation_mgr: AttestationSha384,
    htimedelta: Once<u64>,
}
```

Host/Tellus 的 `Vm` 带有 `guests: Some(Guests<T>)`，所以它可以创建子 TVM。普通 GuestVM 没有 `Guests<T>`，因此不能通过 CoVH 再创建子 VM。

### `Guests<T>` / `GuestVm<T>`

`src/guest_tracking.rs` 中：

- `Guests<T>` 是 Host VM 维护的子 VM 表。
- `GuestVm<T>` 是一个引用计数的 VM 句柄。
- `GuestState` 只有两种状态：
  - `Init`
  - `Running`

Salus 用类型化访问限制接口阶段：

- 初始化期接口使用 `as_initializing_vm()`。
- 运行期接口使用 `as_finalized_vm()`。
- 少数接口使用 `as_any_vm()`。

这使得比如 measured pages、memory region、vCPU 创建只能在 finalize 前执行，而 `TvmCpuRun`、zero/shared page fault-in、block/remove 只能在 finalize 后执行。

### `VmPages<T>`

`src/vm_pages.rs` 中：

```rust
pub struct VmPages<T: GuestStagePagingMode> {
    page_owner_id: PageOwnerId,
    page_tracker: PageTracker,
    tlb_tracker: TlbTracker,
    regions: RwLock<VmRegionList>,
    nesting: usize,
    root: GuestStagePageTable<T>,
    pte_pages: PtePagePool,
    imsic_geometry: Once<GuestImsicGeometry>,
    iommu_context: Once<VmIommuContext>,
}
```

它负责每个 VM 的 GPA 空间、G-stage 页表、区域类型、TLB version、PTE page pool。

### `PageTracker`

`page-tracking` 管全局物理页所有权和状态。CoVH 操作大多会检查或改变 `PageState`：

- `Mapped`
- `Converting`
- `Converted`
- `ConvertedLocked`
- `VmState`
- `Shared`
- `Blocked`
- `BlockedShared`

Host 要把页面给 GuestVM 用，通常必须先把页面从 Host 页表中 invalidate 并转换成 converted pages。

## 常见调用顺序

Tellus 创建并运行 GuestVM 的典型顺序：

```text
TsmGetInfo
TsmConvertPages           # 捐赠 TVM root/state pages
TvmCreate
TsmConvertPages           # 捐赠 page-table pages
AddPageTablePages
TsmConvertPages           # 捐赠 vCPU state pages
TvmCpuCreate
TvmAddMemoryRegion
TsmConvertPages           # 捐赠 measured page backing pages
TvmAddMeasuredPages
Finalize
TvmAddZeroPages           # 可预先补少量 zero pages
loop:
    TvmCpuRun
    按 VM exit 原因:
        TvmAddZeroPages
        TvmAddSharedPages
        TvmBlockPages + TvmInitiateFence + TvmRemovePages
        TvmPromotePage / TvmDemotePage
TvmDestroy
TsmReclaimPages
```

## 接口清单

### `TsmGetInfo`

功能：Host 查询 Salus/TSM 支持的 TVM 创建参数。

Salus 实现：

```rust
fn get_tsm_info(dest_addr, len, active_pages)
```

实现细节：

- 校验 `len >= sizeof(TsmInfo)`。
- 构造 `TsmInfo`：
  - `tsm_state = TsmReady`
  - `tsm_version = 0`
  - `tvm_state_pages = GuestVm::<T>::required_pages()`
  - `tvm_max_vcpus = VM_CPUS_MAX`
  - `tvm_vcpu_state_pages = VmCpus::required_state_pages_per_vcpu()`
- 用 `active_pages.copy_to_guest()` 写回 Host/Tellus GPA。

伪代码：

```text
if len < sizeof(TsmInfo):
    return InvalidParam

info = {
    state = Ready,
    tvm_state_pages = GuestVm.required_pages(),
    max_vcpus = VM_CPUS_MAX,
    vcpu_state_pages = VmCpus.required_state_pages_per_vcpu(),
}
copy_to_host(dest_addr, info)
return sizeof(TsmInfo)
```

### `TsmConvertPages`

功能：Host 将自己 GPA 中的 4 KiB 页面转换成 confidential/converted 状态，准备捐给 GuestVM 或 Salus 内部状态。

Salus 实现：

```rust
fn convert_pages(page_addr, num_pages)
```

后端：

```rust
VmPages::convert_pages()
VmPages::do_convert_pages()
GuestStagePageTable::invalidate_range()
PageTracker::convert_page()
```

实现细节：

- `page_addr` 被解释为当前 Host/Tellus 的 GPA。
- 只支持 4 KiB 页面。
- 要求页面当前在 Host G-stage 中是 mapped RAM。
- Salus invalidate Host 对这些页的映射。
- `PageTracker` 将页面标记为 `Converting(current_tlb_version)`。
- 返回转换的页面数量。

伪代码：

```text
function TsmConvertPages(page_addr, num_pages):
    gpa = host_guest_addr(page_addr)
    invalidated = host_vm.root.invalidate_range(gpa, num_pages * 4K, only_mapped_ram)

    for page in invalidated:
        PageTracker.convert_page(page, current_tlb_version)

    return num_pages
```

### `TsmReclaimPages`

功能：Host 回收之前 converted 但没有被分配给子 VM 的页面。

Salus 实现：

```rust
fn reclaim_pages(page_addr, num_pages)
```

后端：

```rust
VmPages::reclaim_pages()
VmPages::get_converted_pages()
PageTracker::reclaim_page()
map_zero_pages()
```

实现细节：

- 查找 Host GPA 对应的 converted pages。
- 重新 reclaim 成 Host 的 mappable pages。
- 映射回 Host 的地址空间。
- 当前代码 TODO：不支持还没完成 fence 的 converted pages 直接 reclaim。

伪代码：

```text
pages = host_vm.get_converted_pages(page_addr, 4K, num_pages)
mapper = host_vm.map_zero_pages(page_addr, 4K, num_pages)

for page, gpa in zip(pages, gpa_range):
    mapped = PageTracker.reclaim_page(page.clean())
    mapper.map_page(gpa, mapped)

return num_pages
```

### `TsmInitiateFence`

功能：Host 发起针对自己的页表转换/失效的 fence。

Salus 实现：

```rust
fn initiate_fence(active_vcpu)
```

实现细节：

- 调 `self.vm_pages().initiate_fence()` 增加 TLB version。
- 如果 VM 有 IOMMU context，也发 IOMMU fence。
- 对当前 vCPU 调 `active_vcpu.sync_tlb()`。

伪代码：

```text
host_vm.tlb_tracker.increment()
if iommu_context:
    iommu.fence(...)
active_vcpu.sync_tlb()
return 0
```

### `TsmLocalFence`

功能：当前 CPU 执行本地 TLB 同步。

Salus 实现：

```rust
fn local_fence(active_vcpu)
```

实现细节：

- 不增加全局 TLB version。
- 只调用 `active_vcpu.sync_tlb()`。

适用场景：Tellus 在多 CPU 场景中配合 `TsmInitiateFence`，让各 CPU 做本地同步。

## TVM 生命周期接口

### `TvmCreate`

功能：Host 创建一个新的子 TVM/GuestVM。

Salus 实现：

```rust
fn add_guest(params_addr, len, host_active_vcpu)
```

实现细节：

1. 检查当前 VM 是否有 `Guests<T>`。只有 Host/Tellus 有。
2. 从 Host GPA 拷贝 `TvmCreateParams`。
3. 从 Host converted pages 中取：
   - 4 页 root page-table pages。
   - `GuestVm::<T>::required_pages()` 页 GuestVM 状态页。
4. 检查这些页物理连续。
5. 检查 root pages 满足 `T::TOP_LEVEL_ALIGN`。
6. 通过 `PageTracker::add_active_guest()` 分配新的 `PageOwnerId`。
7. 把 root pages 和 state pages assign 给新 GuestVM，状态为 `VmState`。
8. 创建：
   - `GuestStagePageTable`
   - `VmPages`
   - `Vm`
   - `GuestVm`
9. 加入 Host 的 `Guests<T>` 表。
10. 返回 `guest_id`。

伪代码：

```text
params = copy_from_host(params_addr, sizeof(TvmCreateParams))

root_pages = host_vm.get_converted_pages(params.tvm_page_directory_addr, 4K, 4)
state_pages = host_vm.get_converted_pages(params.tvm_state_addr, 4K, GuestVm.required_pages())

require_contiguous(root_pages)
require_aligned(root_pages.base, TOP_LEVEL_ALIGN)
require_contiguous(state_pages)

guest_id = PageTracker.add_active_guest()

root_pages = assign_as_internal_state(root_pages, guest_id)
root = GuestStagePageTable.new(root_pages, guest_id, PageTracker)

vm_pages = VmPages.new(root, host_vm.nesting + 1)
vm = Vm.new(vm_pages, VmCpus.new())

state_pages = assign_as_internal_state(state_pages, guest_id)
guest = GuestVm.new(vm, state_pages)
host.guests.add(guest)

return guest_id
```

### `TvmDestroy`

功能：销毁 GuestVM。

Salus 实现：

```rust
fn destroy_guest(guest_id)
```

实现细节：

- 将 raw `guest_id` 转为 `PageOwnerId`。
- 从 Host 的 `Guests<T>` 删除对应 GuestVM。
- `Guests::remove()` 会检查引用计数，若还有 outstanding reference，则返回 `GuestInUse`。
- `Vm<T>::drop()` 会递归销毁子 guest，并将 active guest 从 `PageTracker` 删除。

伪代码：

```text
guest = host.guests.get(guest_id)
if ref_count(guest) != 1:
    return InvalidParam

host.guests.remove(guest_id)
drop(guest)
return 0
```

### `Finalize`

功能：完成 GuestVM 初始化，使其进入可运行状态。

Salus 实现：

```rust
fn guest_finalize(guest_id, entry_sepc, entry_arg)
```

实现细节：

- 查找 GuestVM。
- 要求状态为 `Init`。
- 调 `GuestVm::finalize()`。
- `Vm::finalize()` 会：
  - power on boot vCPU 0。
  - 设置 entry PC 和 entry arg。
  - 写入 attestation manager 的 EPC/ARG。
  - 检查 IMSIC 地址不缺失/冲突。
  - finalize attestation manager，锁定静态测量。
- `GuestState` 从 `Init` 变为 `Running`。

伪代码：

```text
guest = host.guests.get(guest_id).as_initializing_vm()
guest.vcpu[0].power_on(entry_sepc, entry_arg)
guest.attestation.set_epc(entry_sepc)
guest.attestation.set_arg(entry_arg)
guest.validate_imsic_addrs()
guest.attestation.finalize()
guest.state = Running
return 0
```

## TVM 初始化期内存接口

### `AddPageTablePages`

功能：给 GuestVM 补充中间页表页。

Salus 实现：

```rust
fn guest_add_page_table_pages(guest_id, from_addr, num_pages)
```

实现细节：

- 可对任意状态的 GuestVM 使用 `as_any_vm()`。
- 从 Host converted pages 中取 `num_pages` 页。
- 每页 assign 成 GuestVM 的 `VmState`。
- 放入 GuestVM 的 `PtePagePool`。

伪代码：

```text
pages = host_vm.get_converted_pages(from_addr, 4K, num_pages)
for page in pages:
    pte_page = PageTracker.assign_page_for_internal_state(page.clean(), guest_id)
    guest.vm_pages.add_pte_page(pte_page)
return 0
```

### `TvmAddMemoryRegion`

功能：声明 GuestVM GPA 中一段 confidential memory region。

Salus 实现：

```rust
fn guest_add_memory_region(guest_id, guest_addr, len)
```

实现细节：

- 只能在 GuestVM `Init` 状态调用。
- 把 `guest_addr..guest_addr+len` 加入 `VmRegionList`，类型为 `Confidential`。
- 只是声明区域，不映射具体物理页。

伪代码：

```text
guest = as_initializing_vm(guest_id)
gpa = guest.guest_addr_from_raw(guest_addr)
guest.vm_pages.add_confidential_memory_region(gpa, len)
return 0
```

### `TvmAddMeasuredPages`

功能：把 Host 中的源数据复制到 converted pages，然后作为 measured confidential pages 映射进 GuestVM。

Salus 实现：

```rust
fn guest_add_measured_pages(
    guest_id,
    src_addr,
    dest_addr,
    page_type,
    num_pages,
    guest_addr,
    active_pages,
)
```

实现细节：

- 只能在 GuestVM `Init` 状态调用。
- `dest_addr` 是 Host 提供的 converted backing pages。
- `src_addr` 是 Host 里的源数据地址，通常是 GuestVM image。
- 从 `dest_addr` 获取 converted pages。
- 在 GuestVM 页表中 reserve 目标 PTE。
- 先把 Host 源数据复制进 converted pages，失败则 unlock page 并返回错误。
- 然后 assign 给 GuestVM，映射进 GuestVM GPA。
- `MeasuredPagesMapper::map_page()` 会自动调用：

```rust
measurement.extend_tvm_page(page.as_bytes(), to_addr.bits())
```

即 measured pages 会进入 PCR2 / `TvmPage` 测量。

伪代码：

```text
pages = host_vm.get_converted_pages(dest_addr, page_size, num_pages)
mapper = guest.vm_pages.map_measured_pages(guest_addr, page_size, num_pages)

initialized = []
for page, src_gpa in zip(pages, src_range):
    bytes = copy_from_host(src_gpa, page_size)
    page.initialize(bytes)
    initialized.push(page)

for page, guest_gpa in zip(initialized, guest_range):
    mapped = PageTracker.assign_page_for_mapping(page, guest_id)
    guest.attestation.extend_tvm_page(mapped.bytes, guest_gpa)
    mapper.map_page(guest_gpa, mapped)

return num_pages
```

## TVM 运行期内存接口

### `TvmAddZeroPages`

功能：把 Host converted pages 作为 zero/confidential pages 映射进已 finalized 的 GuestVM。

Salus 实现：

```rust
fn guest_add_zero_pages(guest_id, page_addr, page_type, num_pages, guest_addr)
```

实现细节：

- 只能在 GuestVM `Running` 状态调用。
- `page_addr` 是 Host converted pages。
- `guest_addr` 是 GuestVM GPA。
- `page_type` 转为 `PageSize`，支持 4K/2M 等由 `PageSize::from(page_type)` 决定。
- 目标 GPA 必须位于 GuestVM 的 confidential region。
- assign pages 给 GuestVM，并映射。
- 不做 measured page 扩展。

伪代码：

```text
pages = host_vm.get_converted_pages(page_addr, page_size, num_pages)
mapper = guest.vm_pages.map_zero_pages(guest_addr, page_size, num_pages)

for page, gpa in zip(pages, guest_range):
    mapped = PageTracker.assign_page_for_mapping(page.clean(), guest_id)
    mapper.map_page(gpa, mapped)

return num_pages
```

### `TvmAddSharedPages`

功能：把 Host 当前拥有的 shareable pages 共享映射进 GuestVM。

Salus 实现：

```rust
fn guest_add_shared_pages(guest_id, page_addr, page_type, num_pages, guest_addr)
```

实现细节：

- 只能在 GuestVM `Running` 状态调用。
- 从 Host 页表中查找 mapped/shareable pages。
- 目标 GuestVM GPA 必须位于 `Shared` 或 `SharedRemovable` region。
- `PageTracker::share_page()` 将 Host 页面状态转为/保持 `Shared(refcount)`。
- GuestVM 映射的是 shared mappable page。

伪代码：

```text
pages = host_vm.get_shareable_pages(page_addr, page_size, num_pages)
mapper = guest.vm_pages.map_shared_pages(guest_addr, page_size, num_pages)

for page, guest_gpa in zip(pages, guest_range):
    shared = PageTracker.share_page(page, host_id)
    mapper.map_page(guest_gpa, shared)

return num_pages
```

## vCPU 接口

### `TvmCpuCreate`

功能：为 GuestVM 添加 vCPU。

Salus 实现：

```rust
fn guest_add_vcpu(guest_id, vcpu_id, state_page_addr)
```

实现细节：

- 只能在 GuestVM `Init` 状态调用。
- 从 Host converted pages 取 `VmCpus::required_state_pages_per_vcpu()` 页。
- 要求物理连续。
- assign 成 GuestVM 的 `VmState`。
- 用 `PageBox::new_with()` 在这些页上构造 `VmCpu`。
- 加入 GuestVM 的 `VmCpus`。

伪代码：

```text
pages = host_vm.get_converted_pages(state_page_addr, 4K, required_vcpu_state_pages)
require_contiguous(pages)

vcpu_pages = assign_as_internal_state(pages, guest_id)
vcpu = VmCpu.new(vcpu_id, guest_id)
guest.vcpus.add(PageBox.new_with(vcpu, vcpu_pages))
return 0
```

### `TvmCpuRun`

功能：运行 GuestVM 的指定 vCPU，直到发生 VM exit。

Salus 实现：

```rust
fn guest_run_vcpu(guest_id, vcpu_id, active_vcpu)
```

实现细节：

- 只能在 GuestVM `Running` 状态调用。
- 当前正在执行 CoVH 调用的是 Host/Tellus 的 `active_vcpu`。
- Salus 将这个 Host vCPU 作为 parent context：

```rust
VmCpuParent::HostVm(active_vcpu)
```

- 进入 `guest_vm.run_vcpu(...)`。
- GuestVM exit 时，Salus 将 exit 信息写回 Host 可见位置：
  - 部分 CSR 写到 Host VS CSR。
  - 其他 CSR/GPR 写到 NACL shared memory area。
- 返回值表示 fatal/non-fatal 等运行结果。

伪代码：

```text
guest = as_finalized_vm(guest_id)
return guest.run_vcpu(vcpu_id, parent = HostVm(active_host_vcpu))
```

## 运行期页表维护接口

这些接口主要由 Tellus 在处理 GuestVM exit 后使用，典型场景包括 share/unshare、page fault 后重映射、huge page promote/demote。

### `TvmInitiateFence`

功能：为某个 GuestVM 发起 TLB/IOMMU fence。

Salus 实现：

```rust
fn guest_initiate_fence(guest_id)
```

实现细节：

- 只能对 `Running` GuestVM 调用。
- 调 `guest_vm.vm_pages().initiate_fence()`。
- 增加 GuestVM 的 TLB version。
- 如有 IOMMU context，发 IOMMU fence。
- 代码中有 TODO：当前复用 Guest 自己的 TLB version；若未来支持嵌套并发 invalidation，可能需要 parent/self 两套 TLB version。

伪代码：

```text
guest.vm_pages.tlb_tracker.increment()
if guest has iommu_context:
    iommu.fence(...)
return 0
```

### `TvmBlockPages`

功能：Host 让 GuestVM GPA range 进入 blocked 状态，通常是后续 remove/promote/demote 前的第一步。

Salus 实现：

```rust
fn guest_block_pages(guest_id, guest_addr, len)
```

后端：

```rust
VmPages::block_pages()
GuestStagePageTable::invalidate_range()
PageTracker::block_page()
```

实现细节：

- 只能对 `Running` GuestVM 调用。
- invalidate GuestVM G-stage range。
- 只允许 block 当前 owner 的 `Mapped` RAM，或 parent 仍拥有的 shared page。
- `PageTracker` 将页面转为：
  - `Blocked(tlb_version)`
  - 或 `BlockedShared(refcount, tlb_version)`

伪代码：

```text
invalidated = guest.root.invalidate_range(guest_addr, len, is_blockable_ram)
for page in invalidated:
    PageTracker.block_page(page, guest.current_tlb_version)
return 0
```

### `TvmUnblockPages`

功能：撤销之前的 block，重新 validate 页表 range。

Salus 实现：

```rust
fn guest_unblock_pages(guest_id, guest_addr, len)
```

实现细节：

- 只能对 `Running` GuestVM 调用。
- 调 `GuestStagePageTable::validate_range()`。
- 对每个页调用 `PageTracker::unblock_page()`。

伪代码：

```text
valid_pages = guest.root.validate_range(guest_addr, len, is_blocked_ram)
for page in valid_pages:
    PageTracker.unblock_page(page)
return 0
```

### `TvmRemovePages`

功能：真正从 GuestVM G-stage 页表中移除已经 blocked 且 fence 完成的页面。

Salus 实现：

```rust
fn guest_remove_pages(guest_id, guest_addr, len)
```

后端：

```rust
VmPages::remove_pages()
GuestStagePageTable::unmap_range()
PageTracker::remove_page()
```

实现细节：

- 只能对 `Running` GuestVM 调用。
- 目标 GPA range 必须在：
  - `ConfidentialRemovable`
  - 或 `SharedRemovable`
- 使用 `tlb_tracker.min_version()` 确认 fence 已覆盖相关 invalidation。
- unmap 后调用 `PageTracker::remove_page()`。
- remove 后页面状态可能变回 `Mapped` 或 `Converted`，取决于原状态和所有权关系。

典型序列：

```text
TvmBlockPages
TvmInitiateFence
TvmRemovePages
```

伪代码：

```text
require region in ConfidentialRemovable or SharedRemovable
tlb_version = guest.tlb_tracker.min_version()

unmapped = guest.root.unmap_range(guest_addr, len, is_blocked_at(tlb_version))
for page in unmapped:
    PageTracker.remove_page(page)
return 0
```

### `TvmPromotePage`

功能：将一段小页映射提升成更大的页映射。

Salus 实现：

```rust
fn guest_promote_page(guest_id, guest_addr, page_type)
```

后端：

```rust
VmPages::promote_page()
GuestStagePageTable::promote_page()
PageTracker::unblock_page()
```

实现细节：

- 只能对 `Running` GuestVM 调用。
- `page_type` 表示提升后的目标页大小。
- 目标 range 必须属于 confidential/shared 类 region。
- 相关页必须已经 `Blocked` 且 fence 完成。
- promote 成功后，释放出来的中间页表页会回到 `PtePagePool`。
- 对被 promote 的物理页执行 `unblock_page()`。

典型序列：

```text
TvmBlockPages(range = 2M)
TvmInitiateFence
TvmPromotePage(page_type = Page2M)
```

### `TvmDemotePage`

功能：将一个大页映射拆成一组小页映射。

Salus 实现：

```rust
fn guest_demote_page(guest_id, guest_addr, page_type)
```

后端：

```rust
VmPages::demote_page()
GuestStagePageTable::demote_page()
PageTracker::unblock_page()
```

实现细节：

- 只能对 `Running` GuestVM 调用。
- `page_type` 表示 demote 后的小页大小。
- 目标 range 必须属于 confidential/shared 类 region。
- 相关大页必须已 blocked 且 fence 完成。
- demote 需要从 `PtePagePool` 取一个空闲页表页。
- 如果 PTE pages 不足，会失败。

典型序列：

```text
TvmBlockPages(range = 2M)
TvmInitiateFence
TvmDemotePage(page_type = Page4k)
```

## 接口按 VM 状态分类

初始化前/运行前，也就是 GuestVM `Init` 状态：

- `TvmCreate`
- `AddPageTablePages`
- `TvmAddMemoryRegion`
- `TvmAddMeasuredPages`
- `TvmCpuCreate`
- `Finalize`

运行期，也就是 GuestVM `Running` 状态：

- `TvmCpuRun`
- `TvmAddZeroPages`
- `TvmAddSharedPages`
- `TvmInitiateFence`
- `TvmBlockPages`
- `TvmUnblockPages`
- `TvmPromotePage`
- `TvmDemotePage`
- `TvmRemovePages`
- `TvmDestroy`

Host 自身页面管理：

- `TsmGetInfo`
- `TsmConvertPages`
- `TsmReclaimPages`
- `TsmInitiateFence`
- `TsmLocalFence`

## 错误处理特点

CoVH 接口大多返回 `EcallResult<u64>`，错误会映射成 SBI error。

常见错误来源：

- guest id 无效：`InvalidParam`
- 当前 VM 不是 Host，缺少 `Guests<T>`：`InvalidParam`
- VM 状态不对：`InvalidParam`
- 地址不对齐或页表 range 不合法：通常映射为 `InvalidAddress` 或 `InvalidParam`
- buffer 太小：具体接口返回 `InvalidParam` 或底层错误
- converted pages 不连续：`InvalidAddress`
- PTE pages 不足：由 `VmPages` 错误转 SBI error
- TLB/fence 未完成导致 remove/promote/demote 失败：由 `VmPages` 错误转 SBI error

## 设计要点

### Host 只能提出请求，Salus 负责仲裁

Tellus 不能直接修改 GuestVM G-stage 页表。它只能通过 CoVH 请求 Salus 操作。Salus 会检查：

- 页面是否归 Host 所有。
- 页面是否已经 converted。
- 目标 GuestVM 是否存在。
- GuestVM 是否处于正确状态。
- GPA 是否落在正确 region。
- 页表操作是否满足 block/fence/remove 顺序。

### 页面进入 GuestVM 前必须转换所有权

除了 shared pages，普通 confidential pages、page table pages、vCPU state pages 都要走：

```text
Host Mapped
  -> TsmConvertPages
  -> Converted / ConvertedLocked
  -> assign to GuestVM
  -> VmState 或 Mapped
```

### measured pages 与 zero pages 区别

- `TvmAddMeasuredPages`
  - 初始化期使用。
  - 从 Host source buffer 拷贝内容。
  - 扩展 GuestVM attestation PCR2。
- `TvmAddZeroPages`
  - 运行期使用。
  - 不测量。
  - 常用于处理 GuestVM confidential page fault。

### shared pages 不转移所有权

`TvmAddSharedPages` 使用 `PageTracker::share_page()`，页面仍由 Host 拥有，只是状态变为 `Shared(refcount)`，并映射给 GuestVM。

### block/fence/remove 是安全页表修改的核心序列

对于会移除或改变现有映射的操作，Salus 要求：

```text
block pages
initiate fence
remove/promote/demote
```

这样可以避免旧 TLB 或 IOMMU translation 继续使用已经被重分配或重解释的页面。

## CoVH 接口速查表

| 接口 | 主要用途 | Salus 后端 |
|---|---|---|
| `TsmGetInfo` | 查询 TSM/TVM 参数 | `get_tsm_info` |
| `TsmConvertPages` | Host 页面转 converted | `VmPages::convert_pages` |
| `TsmReclaimPages` | Host reclaim converted pages | `VmPages::reclaim_pages` |
| `TsmInitiateFence` | Host 发起 TLB/IOMMU fence | `VmPages::initiate_fence` |
| `TsmLocalFence` | 当前 CPU 本地 TLB 同步 | `ActiveVmCpu::sync_tlb` |
| `TvmCreate` | 创建 GuestVM | `add_guest` |
| `TvmDestroy` | 销毁 GuestVM | `destroy_guest` |
| `AddPageTablePages` | 给 GuestVM 补 PTE pages | `guest_add_page_table_pages` |
| `TvmAddMemoryRegion` | 声明 confidential GPA region | `guest_add_memory_region` |
| `TvmAddMeasuredPages` | 添加并测量初始页面 | `guest_add_measured_pages` |
| `TvmAddZeroPages` | 添加运行期 zero/confidential pages | `guest_add_zero_pages` |
| `TvmAddSharedPages` | 添加 shared pages | `guest_add_shared_pages` |
| `TvmCpuCreate` | 创建 vCPU | `guest_add_vcpu` |
| `Finalize` | GuestVM 从 Init 进入 Running | `guest_finalize` |
| `TvmCpuRun` | 运行 vCPU | `guest_run_vcpu` |
| `TvmInitiateFence` | GuestVM TLB/IOMMU fence | `guest_initiate_fence` |
| `TvmBlockPages` | invalidate 并 block GuestVM range | `guest_block_pages` |
| `TvmUnblockPages` | 恢复 blocked range | `guest_unblock_pages` |
| `TvmRemovePages` | 移除 blocked range | `guest_remove_pages` |
| `TvmPromotePage` | 小页提升大页 | `guest_promote_page` |
| `TvmDemotePage` | 大页拆成小页 | `guest_demote_page` |

## 总结

Salus 的 CoVH 实现本质上是一个受控的 TVM 管理 API。Tellus 通过它表达“我要创建/装载/运行/维护 GuestVM”，但所有真正危险的操作都在 Salus 内部完成：

- 页面所有权由 `PageTracker` 决定。
- GuestVM GPA 和 G-stage 页表由 `VmPages` / `GuestStagePageTable` 修改。
- VM 生命周期由 `GuestVm` / `GuestState` 限制。
- vCPU 状态由 `VmCpu` / `ActiveVmCpu` 保存恢复。
- 页表变更通过 block/fence/remove 保证 TLB 与 IOMMU 安全。

从实现风格看，CoVH 不是简单的 syscall 转发，而是一组围绕“页面所有权转换 + G-stage 页表构建 + VM 状态机”的安全操作。
