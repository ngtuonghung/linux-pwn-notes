## I. KHÁI NIỆM NỀN TẢNG

### 1. Kernel là gì?

Trong Linux **monolithic kernel**, "kernel" là **toàn bộ code chạy ở quyền cao nhất (ring 0)**, chịu trách nhiệm:[^1]

- Quản lý **memory** (paging, virtual address, page tables)
- Quản lý **process/thread** (task_struct, scheduler)
- Quản lý **syscall**: cầu nối từ userland vào kernel
- Quản lý **thiết bị** qua driver, interrupt

Tất cả những phần này cùng chạy trong **một address space chung của kernel** (monolithic style). Từ góc độ attacker: **bug ở bất kỳ chỗ nào trong vùng này (nhất là driver) có thể = toàn bộ kernel bị chiếm**.[^2][^3][^1]

### 2. Kernel Space vs User Space

Trên x86_64, bộ nhớ ảo được chia thành hai phần chính:[^4]

**Về địa chỉ ảo (Virtual Address):**

- **User space**: vùng virtual address thấp mà process bình thường dùng (code, heap, stack, mmap)
- **Kernel space**: vùng virtual address cao, được map chỉ dùng được khi CPU đang ở **kernel mode**[^5][^6][^7]

**Về quyền truy cập:**

1. **User process không đọc/ghi trực tiếp kernel memory**
    - Nếu ở user mode mà đụng đến địa chỉ thuộc kernel space ⇒ **page fault** (segfault)
    - Mọi tương tác phải đi qua: **syscall → chuyển sang kernel mode → code kernel chạy**[^6][^8][^5]
2. **Kernel nhìn thấy cả user space + kernel space**
    - Khi ở kernel mode (xử lý syscall), kernel có thể truy cập memory của process (dùng copy_from_user/copy_to_user)
3. **Hậu quả crash:**
    - Crash ở **user space**: chỉ die đúng process đó, hệ thống vẫn sống[^5][^6]
    - Crash ở **kernel space**: thường là **kernel panic** ⇒ cả OS sập[^3][^6][^5]
4. **Tại sao tách ra?**
    - **Bảo mật \& ổn định**: lỗi trong app không kéo sập kernel[^7][^9][^6][^5]
    - **Kiểm soát đặc quyền**: chỉ code trong kernel space mới được thực hiện lệnh "privileged"[^8][^6]

### 3. Kernel nằm ở đâu?

**Trên disk:**

- Kernel thường ở file **/boot/vmlinuz-…**
- Khi boot, bootloader (GRUB) load file này vào memory rồi nhảy vào entry point[^1]

**Trong memory (lúc đang chạy):**

- Kernel code + data được **map cố định vào vùng kernel space** của *mọi process*
- Mỗi process có vùng user space riêng, nhưng **share chung một vùng kernel space** (cùng code, cùng core kernel data)[^7][^1]

***

## II. USER MODE, KERNEL MODE VÀ PRIVILEGE LEVELS

### 1. CPU Mode và Ring

Trên x86_64, CPU luôn có **CPL (Current Privilege Level)** - thường gọi là **ring**:

- **Ring 3**: mức thấp → **user mode**
- **Ring 0**: mức cao → **kernel mode**

**"CPU đang ở user mode"** nghĩa là:

- CPL = 3 (code segment có DPL=3)
- Một số lệnh/thao tác **bị cấm**:
    - Ghi vào thanh ghi control (CR0, CR3, CR4)
    - Thực hiện I/O port (`in`, `out`)
    - Đụng tới page có flag "chỉ kernel"

**"CPU đang ở kernel mode"** nghĩa là:

- CPL = 0
- Làm được mọi thứ: đổi page table, tắt SMEP/SMAP, I/O trực tiếp

**Quan trọng:** Đây là **trạng thái phần cứng thực tế** của CPU tại thời điểm nó đang chạy 1 lệnh cụ thể.

### 2. Chuyển đổi giữa User Mode ↔ Kernel Mode

**User → Kernel:**

- `syscall` instruction (Linux 64-bit)
- `int 0x80` (cũ hơn)
- Exception (page fault, divide by zero)
→ Hardware tự: đổi CS, RSP, CPL → nhảy vào địa chỉ trong MSR (`STAR`, `LSTAR`)

**Kernel → User:**

- `sysretq` hoặc `iretq`
→ Trả lại CS=ring3, RSP=user stack, RIP=user RIP

***

## III. KERNEL VS ROOT: SỰ KHÁC BIỆT VỀ QUYỀN HẠN

### 1. Root là gì?

- **Root = một user trong hệ thống Unix/Linux**, có **UID = 0**
- Tất cả check quyền trong kernel kiểu "user này có được làm X không?" thường được code hóa thành "UID == 0?" hoặc "có capability tương ứng không?"
- Root **không tự** có toàn quyền theo nghĩa vật lý; root chỉ có "tất cả quyền mà kernel CHO PHÉP" trong model hiện tại


### 2. So sánh Kernel vs Root (cụ thể)

Có 2 tầng:

**Tầng CPU / phần cứng (ring):**

- Ring 0 (kernel mode)
- Ring 3 (user mode)

**Tầng OS / user:**

- User thường (UID != 0)
- Root (UID = 0)

**Kernel code:**

- Chạy ở: **ring 0**
- Có thể: bypass mọi check, sửa page table, patch code, đổi creds

**Root process:**

- Vẫn chạy ở: **ring 3 (user mode)** phần lớn thời gian
- Chỉ khi gọi syscall thì tạm thời vào ring 0 (kernel), xử lý xong lại quay về ring 3
- Có "tối đa quyền trong model mà kernel định nghĩa": đọc/ghi mọi file, tạo network, kill process, load module (nếu kernel cho)

**Nhưng:**

- Kernel vẫn có thể **giới hạn root** bằng LSM (SELinux, AppArmor), capabilities, seccomp[^10][^11]
- Về mặt thiết kế: **kernel > root**. Root chỉ là "khách VIP" trong luật chơi do kernel đặt ra


### 3. Root Shell vs RCE Kernel

**Root shell:**

- Process userland có UID = 0, chạy ở **ring 3** hầu hết thời gian
- Bị giới hạn bởi: LSM, seccomp, namespaces, cgroup, KPTI, SMEP/SMAP
- Không chạm trực tiếp vào page table, không patch kernel text, không đọc mọi RAM tùy ý
- Gọi `open`, `mount`, `ptrace`, `ioctl`, `insmod` nhưng **tất cả phải đi qua syscall** và bị kernel quyết định "ok/deny"

**RCE trong kernel (ring 0):**

- **Arbitrary code execution / arbitrary ROP trong context ring 0**
- Control RIP bên trong syscall handler/driver
- Có thể gọi trực tiếp internal function: `commit_creds`, `copy_to_user`, `memcpy`
- Có thể ghi vào: `init_task`, `modprobe_path`, `security_ops`, `sys_call_table`, IDT
- Có thể tắt/bypass: SMEP/SMAP, KPTI, rodata, watchdog, LSM hooks

**Khác biệt:**

- **Root** = tối đa quyền trong khuôn khổ luật kernel
- **RCE kernel** = đứng ngoài luật, tự viết luật


### 4. CVE Kernel và LPE

Về kỹ thuật, rất nhiều bug kernel cho bạn **code exec / arbitrary ROP trong kernel** (ring 0). Nhưng về mô tả CVE/exploit, người ta thường chỉ ghi mục tiêu cuối: *"Local privilege escalation to root"* vì:

- Dễ hiểu với sysadmin
- Từ góc nhìn "defender", việc bạn có ROP trong kernel hay chỉ set UID=0 đều = máy bị chiếm

**Chuỗi tấn công thực tế:**
user bình thường → trigger bug kernel → **control RIP / arbitrary R/W trong kernel** → **thao tác gì đó (thường là chỉnh creds)** → kết quả quan sát được: process thành root

Nên: **không phải CVE không đụng RCE kernel**, mà là RCE kernel bị "gói lại" thành mục tiêu observable là LPE lên root.

***

## IV. LAYOUT BỘ NHỚ KERNEL TRÊN X86_64

### 1. Toàn cảnh: Virtual Address 48-bit

Trên x86_64, VA chia thành 2 nửa lớn:

- **Lower half**: các địa chỉ thấp → Linux dùng làm **user space**
- **Upper half**: các địa chỉ cao → Linux dùng làm **kernel space**

Giữa 2 nửa là vùng "non-canonical" (đụng tới là fault).

Mỗi process nhìn thấy **một không gian VA 256 TB**:

- Phần dưới: user riêng của process
- Phần trên: kernel (chung cho mọi process)


### 2. User Space (Lower Half)

Layout quen thuộc trong phần user space:

- Text (code ELF)
- Data, bss
- Heap (malloc)
- mmap region (SO, stack guard)
- User stack

Tất cả chỉ tồn tại trong vùng "lower half".

### 3. Kernel Space (Upper Half)

Trong upper half, Linux map:[^4]

**a) Kernel text/rodata/data:**

- Code của kernel
- Kernel data, rodata

**b) Direct map / physmap:**

- Vùng map gần như toàn bộ RAM vật lý
- Quan hệ: `kernel_va = phys_addr + DIRECT_MAP_OFFSET`
- Kernel có macro:
    - `__va(phys)` → VA trong direct map
    - `__pa(kaddr)` → physical từ kernel addr

**Ý nghĩa:**

- Kernel có thể truy cập *bất kỳ page RAM* bằng VA trong direct map
- Một số bug cho bạn **pointer vào direct map** → có thể suy ra physical, tìm struct khác, có primitive đọc/ghi gần như mọi RAM

**c) vmalloc area:**

- Chỗ cho `vmalloc`, `ioremap`

**d) vmemmap:**

- Struct page array

**e) Modules area:**

- Kernel modules được load vào

**f) Per-CPU data:**

- Data riêng từng CPU

**g) Kernel stacks:**

- Mỗi task có kernel stack riêng

**Quan trọng cho pwn:**

- Mọi process đều có cùng **một mapping kernel** ở phần trên
- Khác nhau của process nằm ở phần **user** (dưới)
- Nếu leak được 1 kernel pointer, có thể reuse từ bất kỳ process nào


### 4. Kernel Stack và Task

**Cấu trúc:**

- Mỗi **task (process/thread)** có:
    - **task_struct** (metadata)
    - **kernel stack** (8 KB / 16 KB tùy config)

**Kernel stack:**

- Là **một block liên tục** trong kernel VA
- Được gắn với task đó
- Không phải user thấy được
- Thường nằm trong vùng mà kernel dễ truy cập/scan/offset từ task_struct

**Khi process gọi syscall:**

1. Đang chạy **user code**
    - CPU ở **ring 3**
    - `RSP` trỏ vào **user stack**
2. Process gọi syscall (vd: `write`)
    - Lệnh `syscall` làm CPU:
        - Nhảy vào entry kernel
        - Đổi sang **ring 0**
        - **Đổi `RSP` sang kernel stack** của task này
    - Mọi `push`, `call` của code kernel đều dùng **kernel stack**
3. Kernel xử lý xong, quay về user
    - `sysretq`/`iretq`
    - CPU đổi lại: CS = ring 3, RSP = user stack

**Quan trọng:**

- **Kernel stack không được tạo mỗi lần syscall**
- Nó được tạo **một lần khi task được tạo**, rồi **re-use** mỗi khi task đó vào kernel
- Khi task sleep/switch, scheduler lưu context rồi chuyển sang task khác → task khác có **kernel stack riêng**


### 5. Direct Map và MMU

**Điểm quan trọng:**

- Direct map chỉ là **một dải VA trong kernel space**, mapping 1–1 tới RAM
- **Các VA khác** (text, heap, stack, vmalloc, user) **vẫn đi qua MMU** như bình thường
- MMU + page tables luôn hoạt động; direct map chỉ là một "kênh" mapping đặc biệt, không phải "bật mode bypass MMU"

**Trong page tables có nhiều vùng mapping:**

- Kernel text → PA A–B
- Modules → PA C–D
- vmalloc → PA rải rác
- Direct map → toàn bộ RAM 1–1

Tất cả đều là entries trong page table. CPU luôn dùng MMU để translate VA → PA.

**Tại sao cần nhiều mapping cho cùng 1 PA?**

- Một page có thể được access qua:
    - VA trong direct map (tiện cho thao tác low-level)
    - VA riêng (được cấp phát bằng `vmalloc`, hoặc map vào user qua `mmap`)
- Cho phép mapping cùng physical page với:
    - Nhiều VA khác nhau
    - Khác quyền (R/O, NX, user vs kernel)
    - Khác cache attribute

**Liên hệ với exploit:**

- Nếu có arbitrary write vào **địa chỉ trong direct map**, bạn có thể ghi vào *rất nhiều thứ khác* trong hệ thống (vì direct map bao phủ gần như toàn bộ RAM)


### 6. Kernel có tự ghi đè code của mình được không?

**Về khả năng hardware:**

- Kernel ở ring 0, có thể: sửa page table, bỏ readonly, map RAM kiểu gì cũng được
→ Về mặt quyền lực, kernel hoàn toàn có thể ghi đè vùng code của chính nó

**Trong thiết kế Linux hiện đại:**

- Vùng chứa **kernel text** (code) thường được:
    - Map read-only
    - Có thể thêm `rodata`, `CONFIG_STRICT_KERNEL_RWX` để ngăn code bị sửa
- Tự ý ghi đè kernel text **bình thường không làm** vì phá tính ổn định, mở cửa cho exploit

**Tuy nhiên:**

- Một số cơ chế hợp pháp vẫn "thay đổi hành vi" kernel:
    - **load/unload module** (thêm code mới)
    - **kprobes, ftrace, livepatch**: chèn hook, patch tại chỗ một số instruction có kiểm soát
- Khi attacker có arbitrary write trong kernel, họ có thể:
    - Sửa page table để cho phép ghi vào text
    - Rồi patch thẳng code, bypass mọi bảo vệ

***

## V. CƠ CHẾ BẢO VỆ: ASLR VÀ KASLR

### 1. ASLR (Address Space Layout Randomization)

**Định nghĩa:**
Ngẫu nhiên hóa layout bộ nhớ **process userland**:[^12][^13]

- Random vị trí: code (PIE binary), heap, stack, mmap region, ld.so, vDSO
- Được kernel điều khiển qua `kernel.randomize_va_space`

**Mục đích:**
Khó đoán địa chỉ để ROP/jump vào gadget trong userland.

**Ai thực hiện:**

- Kernel quyết định layout cho **mỗi process** khi tạo: vị trí stack/heap/mmap

**Tần suất:**

- Mỗi process, mỗi lần chạy


### 2. KASLR (Kernel ASLR)

**Định nghĩa:**
Áp dụng randomization **cho chính kernel**:[^14][^15]

- Random **base address của kernel image** (text, data) trong kernel space
- Thường random luôn: base của kernel text, base vùng modules, đôi khi base của vmalloc, stack, dynamic memory tùy config (`CONFIG_RANDOMIZE_BASE`, `CONFIG_RANDOMIZE_MEMORY`)

**Không random:**

- Từng function (đó là các nghiên cứu nâng cao như FG-KASLR/Chaos)

**Ai thực hiện:**

- Code **trong kernel boot path**:
    - Lúc khởi động, kernel lấy entropy (RDRAND, timer, seed từ bootloader)
    - Chọn **offset ngẫu nhiên** cho: base virtual của kernel text, base physical nơi kernel được đặt, base modules, vmalloc, vmemmap[^15][^16][^14]
- Control bởi config `CONFIG_RANDOMIZE_BASE`, boot param `nokaslr`[^17][^14]

**Tần suất:**

- Mỗi **boot** (base kernel, modules), trừ các cơ chế nghiên cứu kiểu re-randomization runtime như Adelie/Chaos[^18][^19][^20]


### 3. So sánh ASLR vs KASLR

| Aspect | ASLR | KASLR |
| :-- | :-- | :-- |
| **Đối tượng bảo vệ** | Từng process userland | Kernel image + một số vùng kernel |
| **Ai thực hiện** | Kernel quyết định layout cho mỗi process | Kernel tự quyết định nó sẽ được map ở đâu |
| **Tần suất random** | Mỗi process, mỗi lần chạy | Mỗi boot |
| **Control** | `kernel.randomize_va_space` sysctl | `CONFIG_RANDOMIZE_BASE`, `nokaslr` boot param |

### 4. Các vùng Kernel Memory thường được Random

**Commonly randomized (KASLR):**

- Kernel text base (và thus rodata/data relative)
- Modules area base
- vmalloc base
- vmemmap base
- Per-CPU base (arch/config-dependent)

**Not (typically) randomized:**

- Direct map base (fixed pattern)
- Fixmap constants

***

## VI. KHAI THÁC KERNEL

### 1. Bypass KASLR

**"Khắc chế KASLR" nghĩa là gì?**

Trong ngữ cảnh exploit, **bypass/neutralize KASLR** thường = ít nhất một trong:

1. **Leak địa chỉ kernel**
    - Leak pointer `printk`, `task_struct`, `modprobe_path`, function pointer
    - Từ đó suy ra **kernel text base / modules base**
2. **Đưa kernel về trạng thái non-random**
    - Boot với `nokaslr`
    - Bug cho phép chỉnh lại page tables
3. **Tìm/gadget không phụ thuộc base** (ít hơn, kiểu bruteforce/info-independent)

Phổ biến nhất: **info leak → biết base → coi như KASLR chết**.

**KASLR chết ⇒ làm được gì?**

1. **ROP kernel dễ dàng**
    - Biết `kernel_base` ⇒ biết mọi offset: `commit_creds`, `prepare_kernel_cred`, `native_write_cr4`, `switch_task_namespaces`
    - Xây ROP chain ring 0: disable SMEP (`mov cr4`), patch LSM hook, gọi `commit_creds`
2. **Ghi chính xác vào struct/field quan trọng**
    - Ghi vào `modprobe_path`, `init_task.cred`, `security_ops`, `sys_call_table`
    - Không còn kiểu "mò mẫm" trong direct map
3. **Bypass thêm các lớp hardening dựa trên randomness**
    - Khi biết base, có thể tính ra: địa chỉ `current` task, địa chỉ `cred`, địa chỉ `per_cpu` struct

**Ví dụ kiểu bypass KASLR điển hình:**

- Lỗi `copy_to_user`/`copy_from_user` nhầm size → leak struct với pointer kernel
- Lỗi info leak trong `/proc`, `seq_file`, debugfs, ioctl struct
- Đọc `dmesg` (nếu không bị cấm) → log crash/printk chứa pointer
- Đọc `/proc/kallsyms` trên hệ thống debug (nếu không bị restricted)

**Chuỗi exploit thường:**
user bình thường → trigger bug kernel → **info leak** (bypass KASLR) → bug → **arbitrary R/W** (hoặc controlled RIP) → dùng base + R/W để: ROP → `commit_creds(prepare_kernel_cred(0))` hoặc patch `modprobe_path` → chạy binary của bạn với root

### 2. API (Hàm) Nhạy Cảm Trong Kernel

Trong khai thác kernel, "API nhạy cảm" = các hàm mà nếu gọi/ghi được vào thì gần như win.

#### Nhóm 1: Lên quyền (creds)

**Mục tiêu:** Biến process hiện tại thành root một cách "sạch".

- `prepare_kernel_cred(struct task_struct *daemon)`
    - Tạo một struct `cred` mới trong kernel
    - Thường gọi với `NULL` → tạo cred "root-like"
- `commit_creds(struct cred *new)`
    - Gắn `new` vào process hiện tại (`current->cred = new`)
    - Sau khi quay về user, process đó thành root (UID/GID = 0, capabilities full)

**Exploit pattern kinh điển:**

```c
cred = prepare_kernel_cred(NULL);
commit_creds(cred);
```

Chỉ cần **control RIP trong kernel** để gọi được 2 hàm này là gần như chắc chắn có root shell.

#### Nhóm 2: Sửa control regs / bảo vệ

**Mục tiêu:** Tắt SMEP/SMAP/KPTI để cho phép chạy shellcode ở user page, hoặc dễ ROP hơn.

- `native_read_cr4()` / `native_write_cr4()`
    - Đọc/ghi thanh ghi `CR4`
    - `CR4` có bit SMEP/SMAP; tắt chúng = kernel có thể `ret` vào userland, thực thi shellcode ở đó

**Exploit thường:**

1. ROP chain:
    - `mov rax, cr4`
    - `and rax, ~(X86_CR4_SMEP|SMAP)`
    - `mov cr4, rax`
2. Sau đó `ret` thẳng vào shellcode ở user

#### Nhóm 3: Hook / policy (LSM, security)

**Mục tiêu:** Tắt/bypass SELinux/AppArmor/LSM.

- Bảng `security_ops` (hoặc `security_hook_heads` ở kernel mới)
    - Trỏ tới tập các hàm policy (check permission)
    - Ghi đè vào đây → mọi check quyền gần như vô hiệu hoặc luôn "allow"


#### Nhóm 4: Hệ thống / init_task / modprobe_path

**`init_task` và `current`:**

- `init_task` là process 0/1 gốc; từ nó có thể lần ra danh sách process
- Exploit hay: tìm `current` (task_struct của process hiện tại) → từ đó tìm `cred` và chỉnh UID/GID/caps trực tiếp mà không cần gọi `commit_creds`

**`modprobe_path`:**

- Biến global `modprobe_path` (string) chứa đường dẫn `/sbin/modprobe`
- Kỹ thuật kinh điển:

1. Ghi đè `modprobe_path` thành `/tmp/x`
2. Tạo `/tmp/x` (ELF/mô phỏng) chạy code bạn muốn (setuid root, spawn shell)
3. Trigger "unknown binary" → kernel gọi modprobe → thực thi `/tmp/x` với root


#### Nhóm 5: sys_call_table / hook syscall

**Mục tiêu:** Hook/đổi hành vi syscall toàn hệ thống.

- `sys_call_table` là mảng pointer tới handler syscall
    - Nếu bạn ghi được vào đây, thay `__x64_sys_open` bằng handler riêng: có thể log, backdoor, rootkit

Trên hệ thống hardening, bảng này thường nằm trong vùng read-only, nhưng nếu bạn đã có arbitrary write + patch được page table, vẫn sửa được.

#### Nhóm 6: Memory ops trong kernel

**Mục tiêu:** Dùng như primitive copy/R/W mạnh.

- `copy_from_user`, `copy_to_user`, `memcpy`, `memmove`, `memset`
- Khi bạn đã có ROP trong kernel, các hàm này: giúp copy dữ liệu giữa user <-> kernel hoặc nội bộ kernel nhanh, an toàn hơn so với tự viết gadget

**Ví dụ:** Dùng `copy_to_user` để trả kết quả trực tiếp về userland (info leak có kiểm soát).

***

## VII. TÓM TẮT QUAN TRỌNG

### Các khái niệm cốt lõi:

1. **Kernel vs User space**: Không chỉ là dải địa chỉ, mà là phối hợp giữa mapping + bit đặc quyền + mode hiện tại của CPU
2. **Ring/Mode**: Trạng thái phần cứng thực tế của CPU (ring 0 = kernel mode, ring 3 = user mode)
3. **Kernel > Root**: Kernel là code chạy ở ring 0 có quyền thực sự tuyệt đối; Root chỉ là user UID 0 được kernel cho nhiều đặc quyền nhất trong tầng user
4. **RCE kernel > Root shell**: RCE kernel = control ring 0, có thể bypass/patch mọi thứ; Root shell = chỉ là process có đặc quyền cao trong luật do kernel đặt ra
5. **Direct map**: Vùng VA trong kernel space mapping 1–1 tới toàn bộ RAM, giúp kernel truy cập bất kỳ physical page nào dễ dàng
6. **Kernel stack**: Mỗi task có kernel stack riêng, re-use mỗi khi task đó vào kernel mode, không tạo mới mỗi syscall
7. **KASLR vs ASLR**: KASLR random base của kernel mỗi boot; ASLR random layout process mỗi lần chạy

### Lộ trình khai thác kernel điển hình:

1. **Info leak** → bypass KASLR → biết kernel text base
2. **Trigger bug** → arbitrary R/W hoặc control RIP trong kernel
3. **Exploit primitive**:
    - ROP chain: `commit_creds(prepare_kernel_cred(0))`
    - Hoặc: patch `modprobe_path`, ghi `cred` struct
    - Hoặc: tắt SMEP/SMAP → ret to userland shellcode
4. **Return to userland** với quyền root hoặc arbitrary code execution

### Các lưu ý quan trọng cho sec research:

- Kernel bug thường cho **RCE trong ring 0**, nhưng CVE mô tả là "LPE to root" vì dễ hiểu
- Bypass KASLR gần như **bắt buộc** trong kernel pwn hiện đại
- Hiểu layout memory (direct map, kernel stack, modules area) rất quan trọng cho việc xây dựng exploit
- Các API như `commit_creds`, `prepare_kernel_cred`, `modprobe_path` là target chính khi đã có control

[^1]: https://vietnix.vn/linux-kernel/

[^2]: http://arxiv.org/pdf/2404.08716.pdf

[^3]: https://arxiv.org/pdf/1405.5651.pdf

[^4]: https://docs.kernel.org/arch/x86/x86_64/mm.html

[^5]: https://vsudo.net/blog/kernel-space-va-user-space.html

[^6]: https://cloud.z.com/vn/news/kernel/

[^7]: https://viblo.asia/p/cc-memory-layout-2oKLn8Xy4QO

[^8]: https://vietnix.vn/kernel-la-gi/

[^9]: https://blog.lazytrick.com/post/2025-06-26-linux-driver-user-kernel-space/

[^10]: https://manthang.wordpress.com/2010/11/03/quyen-luc-root-va-van-de-an-toan-he-thong/

[^11]: https://whitehat.vn/threads/quyen-luc-root-va-van-de-an-toan-he-thong.7743/

[^12]: https://en.wikipedia.org/wiki/Address_space_layout_randomization

[^13]: https://linux-audit.com/linux-aslr-and-kernelrandomize_va_space-setting/

[^14]: https://docs.kernel.org/security/self-protection.html

[^15]: https://lwn.net/Articles/569635/

[^16]: https://docs.kernel.org/arch/powerpc/kaslr-booke32.html

[^17]: https://docs.kernel.org/admin-guide/kernel-parameters.html

[^18]: https://arxiv.org/pdf/2201.08378.pdf

[^19]: https://dl.acm.org/doi/10.1145/3503222.3507779

[^20]: https://rusnikola.github.io/files/adelie-asplos22.pdf

