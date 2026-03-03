# 2026q1 Homework1 (warmup)

contributed by < `laneser` >

{%hackmd NrmQUGbRQWemgwPfhzXj6g %}

## 探討〈資訊科技詞彙翻譯〉

### render 的語意與 Linux 核心中的使用場景

- [ ] "render" 在電腦圖學語境中為何應強調「如實呈現」？「渲染」一詞喪失什麼關鍵意涵？在 Linux 核心原始程式碼中，"render" 出現在哪些場景、語境又有哪些？翻閱詞典 (善用學校圖書館) 並考據詞源

[〈資訊科技詞彙翻譯〉](https://hackmd.io/@sysprog/it-vocabulary)已說明 render 的核心語意是「如實呈現」，而「渲染」帶有「誇大」意涵，與 rendering 的精確計算本質相悖，「算繪」更為貼切。以下針對 Linux 核心原始碼進行實際搜尋驗證。

#### Linux 核心原始碼中的搜尋結果

搜索 Linux 核心原始碼（v6.19），"render" 出現在 340 個檔案中。逐一檢視每個檔案的上下文後，歸納為以下語境：

| 語境 | 檔案數 | 代表檔案 | 用法 |
|------|--------|---------|------|
| GPU 算繪引擎 | 255 | [`drivers/gpu/drm/i915/intel_uncore.c`](https://github.com/torvalds/linux/blob/v6.19/drivers/gpu/drm/i915/intel_uncore.c) | `FORCEWAKE_RENDER`——算繪引擎電源域管理 |
| 使之成為某狀態 | 41 | [`drivers/media/usb/pvrusb2/pvrusb2-hdw.c`](https://github.com/torvalds/linux/blob/v6.19/drivers/media/usb/pvrusb2/pvrusb2-hdw.c)、[`drivers/input/mouse/alps.c`](https://github.com/torvalds/linux/blob/v6.19/drivers/input/mouse/alps.c) | `pvr2_hdw_render_useless()`、"render it inoperable"、"render unusable" 等 |
| 格式化呈現 | 26 | [`fs/proc/array.c`](https://github.com/torvalds/linux/blob/v6.19/fs/proc/array.c)、[`net/ceph/messenger.c`](https://github.com/torvalds/linux/blob/v6.19/net/ceph/messenger.c) | `render_sigset_t()`、"Nicely render a sockaddr as a string" |
| 影音媒體算繪 | 14 | [`drivers/media/common/v4l2-tpg/v4l2-tpg-core.c`](https://github.com/torvalds/linux/blob/v6.19/drivers/media/common/v4l2-tpg/v4l2-tpg-core.c)、[`sound/soc/codecs/hdac_hdmi.c`](https://github.com/torvalds/linux/blob/v6.19/sound/soc/codecs/hdac_hdmi.c) | "speed up rendering"（影像測試圖樣）、"stream rendering to multiple ports"（音訊輸出） |
| 其他 | 4 | [`arch/m68k/kernel/traps.c`](https://github.com/torvalds/linux/blob/v6.19/arch/m68k/kernel/traps.c) 等 | "surrender" 中的子字串、"it renders wrong"（顯示異常）等邊緣用法 |

大多數（255 個，佔 75%）集中在 GPU/DRM 子系統，"render" 專指 GPU 的算繪引擎硬體。第二大類（41 個，佔 12%）是英文動詞 "render" 的本義——「使之成為某種狀態」，如 "render useless"（使其不可用）、"render inoperable"（使其無法運作），散見於驅動程式和子系統的註解中。格式化呈現（26 個）則是將內部資料結構「如實呈現」為人類可讀格式，如 [`fs/proc/array.c`](https://github.com/torvalds/linux/blob/v6.19/fs/proc/array.c) 的 `render_sigset_t()`——完全不涉及圖學。

值得注意的是，`drivers/media/usb/pvrusb2/` 中的三個檔案雖然位於影音媒體目錄，實際用法卻是 `pvr2_hdw_render_useless()`（使裝置不可用），屬於「使之成為某狀態」而非「影音媒體算繪」。同理，`drivers/gpu/drm/nouveau/dispnv04/hw.h` 雖在 GPU 目錄，其用法是 "renders the extended crtc regs impervious"（使暫存器不可讀寫），也屬於狀態轉換而非算繪引擎。這說明不能僅憑目錄判斷語意，必須看實際上下文。

核心原始碼中 render 的四種主要用法——算繪引擎、狀態轉換、格式化呈現、影音處理——共通點都是「忠實地將 A 轉變為 B」，沒有任何「誇大修飾」的語意。

### constant 與 immutable 的差異

- [ ] 說明 constant 與 immutable 的差異，並探討程式設計中的關鍵考量

[〈資訊科技詞彙翻譯〉](https://hackmd.io/@sysprog/it-vocabulary)指出 constant 是「恆久不變」，immutable 是「在特定語境下不可修改」。以下透過實驗找出 C99 中哪些機制是 constant、哪些是 immutable，以及規格書為何如此區分。

#### 編譯期測試：`case` 標籤與 constant expression

撰寫 [`constant_immutable.c`](https://github.com/laneser/warmup/blob/main/constant_immutable.c)，所有編譯期測試以 `#ifdef TEST_*` 控制、由 Makefile 的 `-D` 旗標驅動，runtime 實驗則在 `main()` 中執行。全部以 GCC 七個最佳化等級（`-O0`、`-Og`、`-O1`、`-Os`、`-O2`、`-O3`、`-Ofast`）測試。

C99 §6.8.4.2 第 3 段規定：

> The expression of each `case` label shall be an **integer constant expression**.

而 §6.6 第 6 段定義 integer constant expression 的允許運算元：

> An integer constant expression shall have integer type and shall only have operands that are **integer constants, enumeration constants, character constants, `sizeof` expressions** whose results are integer constants, and floating constants that are the immediate operands of casts.

列舉中沒有 `const` 變數——因此 `const int N = 10; case N:` 違反標準。透過 Makefile 對 §6.6 列出的每種允許運算元逐一測試 `case` 標籤，並以 `-Wvla -Werror=vla` 偵測 `int arr[n]` 是否被視為 VLA：

```
Compile-time: constant expression in case label (C99 6.6, 6.8.4.2)
  -O0, -Og, -O1, -Os, -O2, -O3, -Ofast     case_define: compiled
  -O0, -Og, -O1, -Os, -O2, -O3, -Ofast     case_enum: compiled
  -O0, -Og, -O1, -Os, -O2, -O3, -Ofast     case_sizeof: compiled
  -O0, -Og, -O1, -Os, -O2, -O3, -Ofast     case_cast_float: compiled
  -O0, -Og, -O1, -Os, -O2, -O3, -Ofast     case_char: compiled
  -O0                                      case_const_int: compile error
  -Og, -O1, -Os, -O2, -O3, -Ofast          case_const_int: compiled (gcc extension)
  -O0, -Og, -O1, -Os, -O2, -O3, -Ofast     arr[const_int]: VLA warning
```

§6.6 認可的運算元（`#define` 替換後的字面值、`enum`、`sizeof`、浮點數轉型、字元字面值）在全部等級都通過。`const int` 則在 `-O0` 被拒絕，`-Og` 以上卻能編譯——原因是 GCC 在啟用最佳化時，常數折疊（constant folding）提前將 `const int N=10` 的值代入，使前端「看到」的是字面值 10 而非變數。這是 **GCC 的延伸行為**，並非 C99 標準允許，加上 `-pedantic-errors` 後全部等級都正確拒絕。

`int arr[n]` 的 VLA 警告不受最佳化影響，七個等級一致觸發——陣列大小的語意檢查在常數折疊之前就完成了。原本預期「編譯錯誤與最佳化等級無關」，但 `case` 標籤的測試證明 GCC 的最佳化階段會回饋影響前端的接受度，印證了跑完全部最佳化等級的必要性。

#### 編譯期測試：mutability 與 addressability

進一步測試這些機制能否被修改、能否取址：

```
Compile-time: mutability and addressability
  -O0, -Og, -O1, -Os, -O2, -O3, -Ofast     enum_assign: compile error (expected)
  -O0, -Og, -O1, -Os, -O2, -O3, -Ofast     enum_addr: compile error (expected)
  -O0, -Og, -O1, -Os, -O2, -O3, -Ofast     const_addr: compiled (expected)
  -O0, -Og, -O1, -Os, -O2, -O3, -Ofast     define_redefine: compiled (expected)
```

| 機制 | 能否更改值 | 能否取址 | 有記憶體位置 |
|------|-----------|---------|------------|
| `#define V 10` | 可以（`#undef` 後重新定義） | 不行（前處理後消失） | 沒有 |
| `enum{V=10}` | 不行（`V=20` → lvalue required） | 不行（`&V` → lvalue required） | 沒有 |
| `sizeof(int)` | 不行（編譯期決定） | 不行（不是物件） | 沒有 |
| `'A'`（字元字面值） | 不行（字面值） | 不行 | 沒有 |
| `const int N=10` | 語意上不行，可透過指標繞過（UB） | **可以** | **有** |

§6.6 認可的運算元有一個共通點：**不佔記憶體、不可取址**，因此不存在被修改的可能——這才是 constant。`const int` 有記憶體位置、可取址，保護是語境相關的（全域有 `.rodata` 硬體保護，區域只有編譯器語意檢查），因此只達到 immutable 的層級。

值得注意的是 `#define`：巨集本身可以 `#undef` 後重新定義，同一個名稱在原始碼不同位置可代表不同值。但這發生在前處理階段——替換完成後，每個使用點都成為字面值（integer constant），在 C 語言層級是 constant。

#### 實驗 1：強制修改 `const` 物件是未定義行為

透過指標強制轉型修改 `const int local = 100` 後，觀察透過變數名稱讀取的值（C99 §6.7.3.5）：

```
Experiment 1: modify const local via cast (UB: C99 6.7.3.5)
  -O0                                      local_name=999 local_ptr=999
  -Og, -O1, -Os, -O2, -O3, -Ofast          local_name=100 local_ptr=999
```

`-O0` 不做最佳化，老實從記憶體讀取，看到被竄改的 999。`-Og` 以上啟用常數傳播（constant propagation），編譯器假設 `const` 物件不會被修改，直接以字面值 100 取代記憶體讀取。同一段原始碼在不同最佳化等級下產生不同結果——這就是未定義行為的具體表現。

#### 實驗 2：全域 `const` 被放入唯讀記憶體

C99 §6.7.3 footnote 114：

> The implementation may place a const object that is not volatile in a read-only region of storage.

透過讀取 `/proc/self/maps` 驗證各變數所在記憶體區段的權限：

```
Experiment 2: memory permissions (C99 6.7.3, footnote 114)
  -O0, -Og, -O1, -Os, -O2, -O3, -Ofast     g_const      perms=r--p writable=no
  -O0, -Og, -O1, -Os, -O2, -O3, -Ofast     g_mutable    perms=rw-p writable=yes
  -O0, -Og, -O1, -Os, -O2, -O3, -Ofast     local_const  perms=rw-p writable=yes
```

全域 `const` 被連結器放入 `.rodata`（唯讀區段），作業系統以頁面保護強制不可寫。區域 `const` 在 stack 上，頁面權限是 `rw-p`——硬體不阻擋寫入，保護只來自編譯器的語意檢查。七個最佳化等級結果一致，因為記憶體配置由連結器決定，與最佳化無關。

> 完整程式碼與 Makefile：[laneser/warmup](https://github.com/laneser/warmup)
> 執行 `make constant_immutable_report` 可重現上述結果。

#### 程式設計中的關鍵考量

綜合編譯期測試和 runtime 實驗的發現，C99 中的「不可變」機制可分為兩個層次：

**Constant（§6.6 認可的 constant expression）：** `enum`、`sizeof`、字元字面值、整數字面值（含 `#define` 替換後的結果）、浮點數轉型。這些不佔記憶體、不可取址，值在編譯期確定且無法被任何手段修改——符合 constant「在所有時空都不變」的語意。

**Immutable（`const` 型別修飾）：** `const` 變數有記憶體位置、可取址。C99 規格書中 "immutable" 一詞完全沒有出現，`const` 的意圖接近 constant——規格書對修改 `const` 物件的描述是 undefined behavior（§6.7.3.5），這已是語言規範所能施加的最強約束。但 C 語言允許直接操作記憶體，使得 `const` 的保護是語境相關的：全域 `const` 有 `.rodata` 硬體保護（實驗 2），區域 `const` 只有編譯器語意檢查（實驗 1）。§6.6 不將 `const` 變數列入 constant expression，正是因為它有記憶體位置，理論上存在被修改的可能。

**`const` 的處境是：意圖接近 constant，但因為 C 語言的低階本質（可取址、可透過指標繞過），實際保護效果只達到 immutable 的層級。** 程式設計者使用 `const` 時，應將其理解為語意合約而非硬體保證，並在需要真正的編譯期常數時使用 `enum` 而非 `const int`。

### concurrent 與 parallel 的語意差異

- [ ] 比較 concurrent 與 parallel 的語意差異，並說明為何「並行」較貼近 concurrent 的本意

[〈資訊科技詞彙翻譯〉](https://hackmd.io/@sysprog/it-vocabulary)已詳述 concurrent（時間重疊，不要求物理同時）與 parallel（物理上同時執行）的區別。查閱教育部[《重編國語辭典修訂本》](https://dict.revised.moe.edu.tw/)後，認同「並行」優於「併發」的理由：「[並行](https://dict.revised.moe.edu.tw/dictView.jsp?ID=19762)」強調**持續進行**，「[併發](https://dict.revised.moe.edu.tw/dictView.jsp?ID=19706)」強調**瞬間發生**——concurrent 的語意是多個執行流持續同時推進，而非同時觸發。

### process, thread, task, job 的區分

- [ ] 當我們撰寫 Linux 核心文件，應如何區分 process, thread, task, job 等術語，才能避免跨領域誤解又省去過多的中英並陳？

這四個術語在不同領域有不同意義，撰寫 Linux 核心文件時應以核心原始碼中的實際用法為準。以下從 [`kernel/fork.c`](https://github.com/torvalds/linux/blob/v6.19/kernel/fork.c)、[`include/linux/sched.h`](https://github.com/torvalds/linux/blob/v6.19/include/linux/sched.h)、[`include/linux/sched/jobctl.h`](https://github.com/torvalds/linux/blob/v6.19/include/linux/sched/jobctl.h) 等檔案歸納各術語在核心中的具體定義。

#### task：核心排程實體的通用結構

`struct task_struct`（定義於 [`include/linux/sched.h`](https://github.com/torvalds/linux/blob/v6.19/include/linux/sched.h)）是核心中**所有可排程實體的通用結構**，無論 process 或 thread 都是一個 `task_struct`。

但核心原始碼中 "task" 主要出現在直接操作 `task_struct` 的場合。面向人類閱讀的註解和函式名稱，多數使用語意更明確的 "process" 或 "thread"。例如 [`wake_up_process()`](https://github.com/torvalds/linux/blob/v6.19/kernel/sched/core.c#L4336) 的註解：

```c
/**
 * wake_up_process - Wake up a specific process
 * @p: The process to be woken up.
 *
 * Attempt to wake up the nominated process and move it to the
 * set of runnable processes.
 */
```

函式名稱叫 `wake_up_process`，註解自然用 "process"——雖然參數型別是 `struct task_struct *`。[`kernel/signal.c`](https://github.com/torvalds/linux/blob/v6.19/kernel/signal.c) 也一樣："Tell a **process** that it has a new active signal"，因為信號傳遞的對象在概念上就是 process。

換言之，核心開發者根據語境選擇最易理解的術語：**明確涉及 `task_struct` 結構操作時用 "task"，描述行為語意時用 "process" 或 "thread"**。這不是術語混亂，而是為了讓原始碼的讀者能快速掌握當下討論的抽象層次。

#### process 與 thread：同一結構，不同旗標

在核心中，process 和 thread 的區別不是結構上的，而是**旗標上的**。兩者都透過 `kernel_clone()` → `copy_process()` → `dup_task_struct()` 建立，差別在 `clone_flags`。以下三段程式碼分布在 [`kernel/fork.c`](https://github.com/torvalds/linux/blob/v6.19/kernel/fork.c) 的不同位置，共同決定 process 與 thread 的區分：

`CLONE_THREAD` 決定是否共享 thread group（[第 2291 行](https://github.com/torvalds/linux/blob/v6.19/kernel/fork.c#L2291)）：

```c
if (clone_flags & CLONE_THREAD) {
    p->group_leader = current->group_leader;
    p->tgid = current->tgid;
} else {
    p->group_leader = p;
    p->tgid = p->pid;
}
```

`exit_signal` 在另一個區塊設定，同時處理 `CLONE_PARENT` 的情況（[第 2370 行](https://github.com/torvalds/linux/blob/v6.19/kernel/fork.c#L2370)）：

```c
if (clone_flags & (CLONE_PARENT|CLONE_THREAD)) {
    p->real_parent = current->real_parent;
    p->parent_exec_id = current->parent_exec_id;
    if (clone_flags & CLONE_THREAD)
        p->exit_signal = -1;
    else
        p->exit_signal = current->group_leader->exit_signal;
} else {
    p->real_parent = current;
    p->parent_exec_id = current->self_exec_id;
    p->exit_signal = args->exit_signal;
}
```

行程計數在 `thread_group_leader()` 判斷後遞增（[第 2418 行](https://github.com/torvalds/linux/blob/v6.19/kernel/fork.c#L2418)）：

```c
if (thread_group_leader(p)) {
    /* ... attach_pid, signal 初始化 ... */
    __this_cpu_inc(process_counts);
} else {
    current->signal->nr_threads++;
    current->signal->quick_threads++;
    atomic_inc(&current->signal->live);
    /* ... */
}
```

而 `thread_group_leader()` 本身定義於 [`include/linux/sched/signal.h`](https://github.com/torvalds/linux/blob/v6.19/include/linux/sched/signal.h#L704)：

```c
static inline bool thread_group_leader(struct task_struct *p)
{
    return p->exit_signal >= 0;
}
```

**process** = thread group leader（`exit_signal >= 0`，`tgid == pid`），擁有獨立的位址空間、檔案描述子表、`signal_struct`，會遞增 `process_counts`。

**thread** = 非 leader 的 task（`exit_signal == -1`），**借用** process 的資源。`pthread_create` 對應的 `clone()` 帶有 `CLONE_VM | CLONE_FILES | CLONE_SIGHAND | CLONE_THREAD` 等旗標，代表共享位址空間、檔案描述子表、信號處理器，不需要複製這些結構——因此建立 thread 的成本遠低於 process。

雖然兩者都是 `task_struct`，但 `clone_flags` 決定了多少資源需要複製、多少可以共享，這才是 process 與 thread 的本質差異。事後辨識則透過 `thread_group_leader()`（檢查 `exit_signal >= 0`）。

#### job：shell 層級的工作控制

"job" 在核心中**不是一種實體**，而是指 job control 機制。[`include/linux/sched/jobctl.h`](https://github.com/torvalds/linux/blob/v6.19/include/linux/sched/jobctl.h) 定義了 `task->jobctl` 的旗標位元：

```c
#define JOBCTL_STOP_SIGMASK    0xffff  /* 最後一次 group stop 的信號編號 */
#define JOBCTL_STOP_PENDING_BIT 17     /* task 應為 group stop 而停止 */
#define JOBCTL_TRAP_STOP_BIT    19     /* 為 STOP 設陷阱 */
```

當 shell 按下 Ctrl-Z 送出 `SIGTSTP`，核心透過 `do_signal_stop()`（[`kernel/signal.c`](https://github.com/torvalds/linux/blob/v6.19/kernel/signal.c)）設定 thread group 中所有 task 的 `JOBCTL_STOP_PENDING`，實現整個 process group 的暫停。"job" 在核心中始終與信號和 process group 綁定，不是獨立的排程單位。

#### 排程器操作的對象：`sched_entity`

延伸觀察：排程器實際操作的**不是 `task_struct`**，而是內嵌其中的 `sched_entity`。CFS 排程器的 `pick_task_fair()`（[`kernel/sched/fair.c`](https://github.com/torvalds/linux/blob/v6.19/kernel/sched/fair.c#L8874)）透過 `pick_next_entity()` 選出 `sched_entity`，再透過 `task_of(se)`（即 `container_of`）轉回 `task_struct`。以下擷取關鍵路徑（省略 group scheduling 迴圈和 throttle 處理）：

```c
static struct task_struct *pick_task_fair(struct rq *rq, struct rq_flags *rf)
{
    struct sched_entity *se;
    /* ... */
    do {
        se = pick_next_entity(rq, cfs_rq);
        cfs_rq = group_cfs_rq(se);
    } while (cfs_rq);

    p = task_of(se);
    return p;
}
```

`task_struct` 透過組合（composition）包含多種排程實體（`sched_entity`、`sched_rt_entity`、`sched_dl_entity`），但排程器只操作對應的子結構。

#### 官方文件中的術語佐證

以上分析來自原始碼，以下從 [docs.kernel.org](https://docs.kernel.org/) 的官方文件交叉驗證這些觀察。核心文件中沒有統一的 glossary 頁面定義這四個術語，但多處文件的用法一致，可作為佐證。

**task = 排程實體的通稱。** [CFS Scheduler](https://docs.kernel.org/scheduler/sched-design-CFS.html) 文件通篇以 "task" 指稱排程單位，並在提到 scheduling entity 時直接等同：

> "It puts the scheduling entity (task) into the red-black tree and increments the nr_running variable."

[SCHED_DEADLINE](https://docs.kernel.org/scheduler/sched-deadline.html) 文件同樣以 "task" 作為排程實體的通稱，不區分 process 或 thread：

> "A SCHED_DEADLINE task should receive 'runtime' microseconds of execution time every 'period' microseconds"

這與原始碼分析一致：排程器層級只認 task（`sched_entity`），不關心它是 process 還是 thread。

**PID 對應 thread，TGID 對應 process（thread group）。** [ftrace](https://docs.kernel.org/trace/ftrace.html) 文件在描述 `record-tgid` 選項時，明確定義了 PID 與 TGID 的對應關係：

> "on each scheduling context switch the Task Group ID of a task is saved in a table mapping the PID of the thread to its TGID."

這段文字的術語選擇非常精確：PID 屬於 "thread"，TGID（Task Group ID）屬於 thread group。對照原始碼中 `fork.c` 的 `p->tgid = current->tgid`（`CLONE_THREAD` 時共享 tgid）和 `p->tgid = p->pid`（新 process 時 tgid 等於自身 pid），文件與實作完全吻合——process 就是 thread group leader，其 pid 即為整個 group 的 tgid。

**`fork()` 建立的是 thread（排程層級）。** [sysctl 文件](https://docs.kernel.org/admin-guide/sysctl/kernel.html)中 `threads-max` 的描述：

> "the maximum number of threads that can be created using fork()"

這裡把 `fork()` 建立的東西稱為 "thread"，乍看似乎與一般認知（`fork()` 建立 process）矛盾。但從核心排程器的視角，每個 `task_struct` 都是一個排程執行緒；`fork()` 建立的新 process 同時也是一個新的排程執行緒（只是它恰好是 thread group leader）。`threads-max` 計算的是系統中所有 `task_struct` 的數量上限，不分 process 或 thread。

#### 撰寫核心文件時的術語選擇

| 術語 | 核心中的意義 | 中文翻譯 |
|------|------------|---------|
| task | `task_struct`，所有可排程實體的通用稱呼 | 直接用 task（無精確對應中文） |
| process | thread group leader（`exit_signal >= 0`） | 行程 |
| thread | 非 leader 的 task（`CLONE_THREAD`） | 執行緒 |
| job | job control 機制，非獨立實體 | 直接用 job control（中文「工作控制」可用但少見） |

"task" 和 "job" 在核心語境中有特定技術含義，直接使用英文原文較不易產生跨領域混淆。"process" 和 "thread" 可用「行程」「執行緒」，但須意識到核心中兩者共用同一個 `task_struct` 結構，差異在於 `clone_flags` 決定的資源共享程度——thread 借用 process 的位址空間和檔案描述子，建立成本較低。

## 細讀〈GNU/Linux 開發工具〉

### 開發環境配置

- [ ] 確保 Linux 系統已正確安裝，且每天使用

目前使用三機架構進行開發與測試：

| 角色 | 硬體 | OS | 用途 |
|------|------|-----|------|
| 主力開發機 | AMD Ryzen 7 9800X3D 8C/16T，16 GiB RAM | Ubuntu 24.04.4 LTS（Docker on WSL2） | 編輯、AI 協作、git 操作 |
| ARM 實體機 (`lab0`) | Raspberry Pi 3 Model B v1.2，Cortex-A53 4C，906 MiB RAM | Debian 13 (trixie)，核心 6.12.62-v8（aarch64） | 跨架構編譯測試 |
| x86 實體機 (`lab-x86`) | Intel Celeron J1800 2C，3.7 GiB RAM | Ubuntu 24.04.1 LTS，核心 6.12.74-patched+（x86_64） | 原生編譯、valgrind、perf |

日常工作流程：在 Docker container 內透過 VS Code Dev Container 開發，需要原生測試時透過 SSH 同步至實體機：

```
rsync -avz --exclude='.git' homework/lab0-c/ lane@lab0:~/lab0-c/
ssh lane@lab0 'cd ~/lab0-c && make && make test'
```

兩台實體機分別提供 ARM 和 x86 環境，可驗證程式在不同架構下的行為差異（如 alignment 要求、endianness、`sizeof` 差異等）。

### HackMD 與 LaTeX

- [ ] 學習 HackMD 的操作，並可正確使用 LaTeX。能夠在 HackMD 筆記中用 LaTeX 語法展現波函數和密度矩陣

HackMD 支援行內公式（`$...$`）和獨立公式（`$$...$$`）。以下整理學習 LaTeX 過程中掌握的語法，並以波函數和密度矩陣為例展示。

#### 常用 LaTeX 語法

| 語法 | 效果 | 說明 |
|------|------|------|
| `x^2` | $x^2$ | 上標（指數） |
| `x_n` | $x_n$ | 下標 |
| `\frac{a}{b}` | $\frac{a}{b}$ | 分數，第一個 `{}` 放分子，第二個放分母 |
| `\sum_{i=1}^{N}` | $\sum_{i=1}^{N}$ | 求和符號 |
| `\partial` | $\partial$ | 偏微分 |
| `\hbar` | $\hbar$ | 約化 Planck 常數（$\hbar = \frac{h}{2\pi}$） |
| `\Psi`, `\psi`, `\rho` | $\Psi$, $\psi$, $\rho$ | 希臘字母 |
| `\langle`, `\rangle` | $\langle$, $\rangle$ | 狄拉克（Dirac）bra-ket 記號的角括號 |
| `\begin{pmatrix} a & b \\ c & d \end{pmatrix}` | $\begin{pmatrix} a & b \\ c & d \end{pmatrix}$ | 矩陣，`&` 分欄，`\\` 換行 |

#### 波函數

波函數 $\Psi(x, t)$ 描述量子系統的狀態。$\Psi$ 本身是複數值函式，不直接是機率；$|\Psi(x,t)|^2$ 才是機率密度——在 $[a, b]$ 區間找到粒子的機率為：

$$P(a \leq x \leq b) = \int_a^b |\Psi(x,t)|^2 \, dx$$

歸一化條件要求粒子一定在某處：

$$\int_{-\infty}^{\infty} |\Psi(x,t)|^2 \, dx = 1$$

三維空間中，波函數寫為 $\Psi(\mathbf{r}, t)$，其中 $\mathbf{r} = (x, y, z)$。波函數的時間演化由薛丁格（Schrödinger）方程式決定：

$$i\hbar \frac{\partial}{\partial t} \Psi(\mathbf{r}, t) = \hat{H} \Psi(\mathbf{r}, t)$$

左邊是 $i \cdot \hbar$ 乘以波函數對時間的偏微分（波函數隨時間的變化率），右邊是 Hamilton 運算子 $\hat{H}$（代表系統總能量）作用在波函數上。白話說：波函數怎麼隨時間演化，由系統的能量結構決定。

#### 密度矩陣

波函數只能描述**純態**（確定處於某個量子態）。當系統的狀態不確定——例如只知道「70% 機率處於 $|0\rangle$，30% 機率處於 $|1\rangle$」——這種**混合態**需要密度矩陣來描述。密度矩陣由[馮·諾伊曼](https://zh.wikipedia.org/wiki/%E7%BA%A6%E7%BF%B0%C2%B7%E5%86%AF%C2%B7%E8%AF%BA%E4%BC%8A%E6%9B%BC)（von Neumann）與[列夫·朗道](https://zh.wikipedia.org/wiki/%E5%88%97%E5%A4%AB%C2%B7%E6%9C%97%E9%81%93)（Landau）於 1927 年分別獨立提出。

這裡的 $|0\rangle$（ket）和 $\langle 0|$（bra）是[狄拉克符號](https://zh.wikipedia.org/wiki/%E7%8B%84%E6%8B%89%E5%85%8B%E7%AC%A6%E5%8F%B7)，分別代表行向量和列向量：

$$|0\rangle = \begin{pmatrix} 1 \\ 0 \end{pmatrix}, \quad \langle 0| = \begin{pmatrix} 1 & 0 \end{pmatrix}$$

ket 乘以 bra（外積）得到矩陣，這就是純態的密度矩陣：

$$|0\rangle\langle 0| = \begin{pmatrix} 1 \\ 0 \end{pmatrix} \begin{pmatrix} 1 & 0 \end{pmatrix} = \begin{pmatrix} 1 & 0 \\ 0 & 0 \end{pmatrix}$$

混合態「70% 在 $|0\rangle$，30% 在 $|1\rangle$」的密度矩陣，就是各純態密度矩陣的加權和：

$$\rho = 0.7 |0\rangle\langle 0| + 0.3 |1\rangle\langle 1| = \begin{pmatrix} 0.7 & 0 \\ 0 & 0.3 \end{pmatrix}$$

一般化的混合態密度矩陣定義為：

$$\rho = \sum_i p_i |\psi_i\rangle\langle\psi_i|, \quad \sum_i p_i = 1, \quad p_i \geq 0$$

純態與混合態的差別可透過密度矩陣的非對角元素區分。疊加態 $|\psi\rangle = \alpha|0\rangle + \beta|1\rangle$ 的密度矩陣含有非零的 $\alpha\beta^*$ 項（coherence），代表量子干涉效應；混合態的 coherence 消失，對角線只剩古典機率。

### Git 操作

- [ ] 學習 git 的操作，依據給定的教學錄影實際練習，並知曉 `git rebase` 一類命令的操作

#### `git rebase` 的運作原理

`git merge` 保留完整的分支歷史，產生 merge commit；`git rebase` 則將一系列 commit 重新套用（replay）到新的基底上，產生線性歷史。以下圖示說明差異：

```
# merge 前
main:    A---B---C
              \
feature:       D---E

# git merge（保留分支結構）
main:    A---B---C---M
              \     /
feature:       D---E

# git rebase（線性化）
main:    A---B---C
                  \
feature:           D'---E'
```

`git rebase main` 的實際步驟：
1. 找出 `feature` 分支與 `main` 的共同祖先（B）
2. 將 B 之後的 commit（D、E）暫存為 patch
3. 將 `feature` 的 HEAD 重置到 `main` 的最新 commit（C）
4. 依序將暫存的 patch 重新套用，產生新的 commit（D'、E'）

D' 和 E' 的內容與 D、E 相同，但 commit hash 不同——因為 parent commit 改變了。

#### Interactive rebase

`git rebase -i HEAD~3` 可對最近 3 個 commit 進行互動式操作：

| 指令 | 作用 |
|------|------|
| `pick` | 保留該 commit |
| `reword` | 保留但修改 commit message |
| `squash` | 與前一個 commit 合併，保留兩者的 message |
| `fixup` | 與前一個 commit 合併，丟棄此 commit 的 message |
| `drop` | 刪除該 commit |

在課程作業中，interactive rebase 適合用來整理開發過程中的瑣碎 commit（如 "fix typo"、"debug print"），在提交前合併為語意完整的 commit。

#### 使用限制

`git rebase` 改寫歷史，因此有一條基本原則：**不對已推送到共享分支的 commit 執行 rebase**。若其他人已基於這些 commit 開發，rebase 會導致歷史分歧，造成合併困難。本地尚未推送的分支則可自由 rebase。

## 探討〈解讀計算機編碼〉

### 固定位元寬度的加法等價於 $\mathbb{Z}/2^k\mathbb{Z}$

- [ ] 為何計算機的加法在固定 $k$ 位元下，本質等價於 $\mathbb{Z}/2^k\mathbb{Z}$ 上的加法？

$k$ 位元能表示的值恰好是 $\{0, 1, 2, \ldots, 2^k - 1\}$，共 $2^k$ 個元素。二進位加法若結果超出 $k$ 位元，溢出的高位元被硬體捨棄——這在數學上等於 $\bmod\ 2^k$。

以 $k = 3$ 為例，集合為 $\{0, 1, \ldots, 7\}$，$6 + 5 = 11$，硬體運算為 `110 + 101 = [1]011`，第 4 位元溢出被丟棄，結果為 `011` = 3，即 $11 \bmod 8 = 3$。

這不是巧合，而是硬體的物理限制（只有 $k$ 條線路）自動實現了模除運算。因此 $k$ 位元無號加法與 $\mathbb{Z}/2^k\mathbb{Z}$ 的加法是同一件事——集合相同（$2^k$ 個元素），運算規則相同（加完取 $\bmod 2^k$）。教材以「圓環」比喻：$\mathbb{Z}/2^k\mathbb{Z}$ 是一個有 $2^k$ 個刻度的時鐘，加法是順時針轉圓弧，繞過 $2^k - 1$ 之後回到 $0$。

### 允許溢位保證封閉性

- [ ] 為何「允許溢位」反而保證封閉性？若不允許溢位，會破壞哪些群的性質？

若不允許溢位（例如溢位時報錯中斷），以 $k = 3$ 的集合 $\{0, \ldots, 7\}$ 和普通整數加法來看，阿貝爾群的五項性質中只剩單位元素（$0$）不受影響：

| 性質 | 不允許溢位 | 允許溢位（$\bmod 2^k$） |
|------|-----------|----------------------|
| 封閉性 | 破壞：$6 + 5 = 11 \notin \{0, \ldots, 7\}$ | 保證：結果自動落回集合 |
| 結合律 | 運算不完備，無法對所有元素成立 | 保證 |
| 單位元素 | $0$ 仍有效 | $0$ 仍有效 |
| 反元素 | 不存在：$5$ 的反元素需 $-5$，但 $-5 \notin \{0, \ldots, 7\}$ | 存在：$-a \equiv 2^k - a$（如 $-5 \equiv 3$） |
| 交換律 | 運算不完備時無意義 | 保證 |

反元素的問題比封閉性更根本——不是溢位導致計算中斷，而是反元素**根本不存在於集合中**。正是允許溢位（$\bmod 2^k$）才**創造**了反元素：$5 + 3 = 8 \equiv 0 \pmod{8}$，使 $3$ 成為 $5$ 的反元素。溢位使得二補數的 $-a$ 可定義為 $\neg a + 1$（位元反轉再加 1），也就是 $2^k - a$。

### 費馬小定理與模反元素

- [ ] 在質數模數 $p$ 下，可用 $a^{-1} \equiv a^{p-2} \pmod p$ 計算反元素，解釋該式如何來自有限群理論，並說明為何此性質僅在「非零元素形成群」時成立。從 Linux 核心實作角度分析: 1) 為何必須確保輸入元素為單位？2) 若傳入零因子會造成什麼風險？

#### 從群論推導費馬小定理

教材（〈[解讀計算機編碼](https://hackmd.io/@sysprog/binary-representation)〉）展示 $\{1, 2, 3, 4\}$ 在 $\bmod 5$（質數）下的乘法表：

| $\times$ | 1 | 2 | 3 | 4 |
|----------|---|---|---|---|
| 1 | 1 | 2 | 3 | 4 |
| 2 | 2 | 4 | 1 | 3 |
| 3 | 3 | 1 | 4 | 2 |
| 4 | 4 | 3 | 2 | 1 |

每行每列都是 $\{1, 2, 3, 4\}$ 的排列（Latin square），代表每個元素都有乘法反元素——這就是群。這個 Latin square 性質是費馬小定理的證明基礎：

取集合 $S = \{1, 2, \ldots, p-1\}$，將每個元素乘以 $a$（$a \not\equiv 0$），得到 $\{a, 2a, \ldots, (p-1)a\} \pmod{p}$。因為乘以 $a$ 是排列（Latin square 性質），這個集合恰好是 $S$ 的重排——因數完全相同，只是順序不同。由交換律，兩邊全部乘起來的結果相等：

$$(p-1)! \cdot a^{p-1} \equiv (p-1)! \pmod{p}$$

左邊提出 $a^{p-1}$ 用到結合律和交換律。$(p-1)!$ 的每個因數都在群中，反元素存在，兩邊乘以 $(p-1)!$ 的反元素消去，得到費馬小定理：

$$a^{p-1} \equiv 1 \pmod{p}$$

將 $a^{p-1}$ 拆成 $a \cdot a^{p-2}$：既然 $a \cdot a^{p-2} \equiv 1$，那麼 $a^{p-2}$ 就是 $a$ 的乘法反元素，即 $a^{-1} \equiv a^{p-2} \pmod{p}$。

#### 僅在非零元素成群時成立

上述證明的每一步都依賴群的性質：封閉性（乘以 $a$ 的結果落回集合）、交換律（因數順序不影響乘積）、反元素存在（消掉 $(p-1)!$）。若模數不是質數，這些性質就不成立。

以 $\bmod 4$（非質數）下 $\{1, 2, 3\}$ 的乘法表為例：

| $\times$ | 1 | 2 | 3 |
|----------|---|---|---|
| 1 | 1 | 2 | 3 |
| 2 | 2 | 0 | 2 |
| 3 | 3 | 2 | 1 |

第 2 行是 $\{2, 0, 2\}$——$2 \times 2 \equiv 0 \pmod{4}$，結果落在集合 $\{1, 2, 3\}$ 之外，封閉性不成立，不是排列。證明的第一步（「乘以 $a$ 是排列」）就失敗了，整個推導鏈斷裂，費馬小定理不適用，$a^{p-2}$ 也就無法作為反元素。

#### Linux 核心中的模反元素

**搜尋思路：** 題目問的是 $a^{-1} \pmod{p}$，程式中稱為 modular inverse。C 語言用 snake_case 命名，先搜尋最直覺的 `mod_inv`、`vli_mod_inv` 等變體。進一步擴大搜尋範圍，用 `fermat`、`inverse`、`p - 2`、`mpi.*inv` 等關鍵字掃過整個 `crypto/` 子系統，確認是否有其他實作。

搜尋結果：

- `fermat`——僅出現在 `drivers/net/usb/zaurus.c` 的 "fermat packet mangling"，與數學無關
- `mod_inverse` / `modular_inverse`——無符合結果
- `p - 2`（在 `crypto/`）——僅 `crypto/dh.c` 的範圍檢查 `2 <= y <= p - 2`，不是在計算 $a^{p-2}$
- `mpi.*inv`——找到 `crypto/rsa.c` 的 `key->qinv`（見下方）

核心中**沒有直接用 $a^{p-2}$ 計算反元素的程式碼**——都採用擴展歐幾里得演算法，效率更高（多項式時間 vs 指數冪次）。找到兩處模反元素的使用：

**橢圓曲線密碼：** [`crypto/ecc.c`](https://github.com/torvalds/linux/blob/v6.19/crypto/ecc.c) 的 `vli_mod_inv()`（[第 1039 行](https://github.com/torvalds/linux/blob/v6.19/crypto/ecc.c#L1039)），runtime 計算模反元素，用於 ECDSA 簽章驗證和點座標轉換。

**RSA CRT 加速：** [`crypto/rsa.c`](https://github.com/torvalds/linux/blob/v6.19/crypto/rsa.c) 的 `key->qinv`（[第 350 行](https://github.com/torvalds/linux/blob/v6.19/crypto/rsa.c#L350)），即 $q^{-1} \bmod p$，在金鑰生成時預先計算並儲存於金鑰結構中，用於中國剩餘定理（CRT）加速 RSA 私鑰運算（[第 102 行](https://github.com/torvalds/linux/blob/v6.19/crypto/rsa.c#L102)）。

兩者的數學前提相同：輸入必須是乘法群的元素（非零且與模數互質），反元素才存在。

**為何必須確保輸入為單位？** 觀察 `vli_mod_inv` 對零輸入的處理（[第 1047 行](https://github.com/torvalds/linux/blob/v6.19/crypto/ecc.c#L1047)）：

```c
if (vli_is_zero(input, ndigits)) {
    vli_clear(result, ndigits);
    return;
}
```

模反元素在數學上是**部分函式**（partial function）——零和與模數不互質的元素沒有定義。但 `vli_mod_inv` 的回傳型別是 `void`，沒有管道回報「此輸入無解」，遇到零輸入只能靜默回傳零。

**若傳入零會造成什麼風險？** 以 ECDSA 簽章驗證（[`crypto/ecdsa.c`](https://github.com/torvalds/linux/blob/v6.19/crypto/ecdsa.c)）為例，驗證流程需要計算 $s^{-1} \bmod n$。呼叫端在計算反元素之前先檢查輸入（[第 34 行](https://github.com/torvalds/linux/blob/v6.19/crypto/ecdsa.c#L34)）：

```c
/* 0 < r < n  and 0 < s < n */
if (vli_is_zero(r, ndigits) || vli_cmp(r, curve->n, ndigits) >= 0 ||
    vli_is_zero(s, ndigits) || vli_cmp(s, curve->n, ndigits) >= 0)
    return -EBADMSG;
```

確認 $s \neq 0$ 且 $s < n$ 後才呼叫 `vli_mod_inv`（[第 44 行](https://github.com/torvalds/linux/blob/v6.19/crypto/ecdsa.c#L44)）。若此檢查被省略，$s = 0$ 會使 $s^{-1}$ 得到 $0$，後續 $u_1 = hash \cdot 0 = 0$、$u_2 = r \cdot 0 = 0$，驗證退化為檢查無窮遠點，簽章驗證邏輯被繞過——攻擊者可偽造任意訊息的合法簽章。

**設計觀察：** `vli_mod_inv` 內部已經檢查了 `vli_is_zero(input)`，但因為回傳 `void`，檢查到了也無法回報，只能靜默回傳零。呼叫端為了安全，又得在呼叫前再做一次零檢查。結果是：若呼叫者有檢查（如 ECDSA），底層的零檢查永遠不會觸發，是多餘的；若呼叫者沒檢查，底層雖然擋住了零但無法通知，等於白查。兩種情境都造成浪費。若函式改為回傳 `int` 錯誤碼，底層一次檢查就能同時完成偵測和通知，呼叫端只需檢查回傳值，無需重複驗證輸入。

不過，回到 ECDSA 的實際情境，即使 `vli_mod_inv` 改為回傳錯誤碼，能省掉的也只有 `vli_is_zero(s)` 這一項。ECDSA 的四個檢查中，`r == 0`、`r >= n` 與 `vli_mod_inv` 完全無關（`r` 不是它的輸入），而 `s >= n` 是 ECDSA 協定層級的約束——`vli_mod_inv` 的擴展 GCD 在 $s \geq n$ 時仍會算出結果（等效於 $(s \bmod n)$ 的反元素），數學上不算錯，但不符合規格要求。因此，`void` 設計的原則性缺陷（部分函式無法回報錯誤）仍然存在，但在此案例中實際影響有限，呼叫端的多數檢查屬於協定層級的驗證，無論如何都不能省略。

### 位元遮罩取餘數僅對 unsigned 安全

- [ ] 為何 $x \% 2^n \equiv x \mathbin{\&} (2^n - 1)$ 僅對 unsigned 或非負整數安全？從 CVE/CWE 找到相關資訊安全的議題

> 注意：原始題目中的 LaTeX 公式 `$x % 2^n \equiv x & (2^n - 1)$` 在 HackMD 上只會算繪出 $x$，因為 `%` 在 LaTeX 中是註解符號（吞掉後面整行），`&` 是表格對齊符號。需切換至編輯模式閱讀 raw markdown 才能看到完整公式。正確寫法應為 `$x \% 2^n \equiv x \mathbin{\&} (2^n - 1)$`。

#### 等價性只在非負時成立

$2^n - 1$ 的二進位表示是 $n$ 個 1（例如 $n = 3$ 時為 `111` = 7）。$x \mathbin{\&} (2^n - 1)$ 的效果是清除高位元、只保留最低 $n$ 位元——對 unsigned 整數來說，這與 $x \% 2^n$ 完全等價。

但對 signed 負數，兩者結果不同。以 $x = -1$（32-bit signed int）、$n = 3$ 為例：

- $(-1) \% 8 = -1$（C99 §6.5.5：餘數的符號與被除數相同）
- $(-1) \mathbin{\&} 7 = 7$（$-1$ 的二補數全為 1，AND 後保留最低 3 位元）

位元 AND 永遠產生非負結果（高位元被清零，包括符號位元），等於丟失了符號資訊。若程式將 $x \mathbin{\&} mask$ 當作 $x \% divisor$ 的最佳化替代，遇到負數輸入就會得到錯誤結果。

#### CWE 與 CVE

題目要求「從 CVE/CWE 找到相關資訊安全的議題」。過去只知道 CVE（Common Vulnerabilities and Exposures），查了才發現 CWE（Common Weakness Enumeration）是不同層次的東西：

- **CWE** 是由 [MITRE](https://cwe.mitre.org/) 維護的**軟體弱點分類系統**，描述的是「這一類錯誤的通用模式」。靜態分析工具（Coverity、Clang Static Analyzer）用 CWE 編號標記問題，開發者用它作為常見錯誤的檢查清單。
- **CVE** 是「某個特定產品中的具體漏洞」，每個 CVE 會標注對應的 CWE 分類，說明根因屬於哪一類。

換言之，**CWE 是病名，CVE 是病例**。

#### CWE 分類與 CERT C 規則

對 signed 值使用位元運算子（包含用 `&` 替代 `%`）的問題，對應的業界規範是 [SEI CERT C 的 INT13-C 規則](https://wiki.sei.cmu.edu/confluence/display/c/INT13-C.+Use+bitwise+operators+only+on+unsigned+operands)：**「位元運算子只應用於 unsigned 運算元」**。該規則指出，對 signed 值做位元運算的結果是 implementation-defined（右移）或可能導致非預期行為（AND、OR 等），可能造成緩衝區溢位甚至任意程式碼執行。

INT13-C 官方映射的 CWE 是 **[CWE-682: Incorrect Calculation](https://cwe.mitre.org/data/definitions/682.html)**——「產生不正確或非預期結果的計算」。這正是 `x & (2^n - 1)` 替代 `x % 2^n` 對 signed 負數算錯的情形：程式設計師假設兩者等價，但對負數輸入 `& mask` 會產生錯誤的非負結果。`x & mask` 中兩個運算元和結果都是 `int`，沒有發生型別轉換，問題純粹是計算本身對負數不成立。

#### 相關 CVE

以各種關鍵字（bitmask modulo signed、signedness bitwise、mask negative 等）搜尋 NVD 和 CWE-682 底下的 CVE，**找不到直接因「`& mask` 替代 `% mod` 對 signed 值算錯」而產生的漏洞**。

#### 核心中的 bitmask-as-modulo 實例

Linux 核心的 [`include/linux/circ_buf.h`](https://github.com/torvalds/linux/blob/v6.19/include/linux/circ_buf.h) 是此 pattern 的典型使用：

```c
struct circ_buf {
    char *buf;
    int head;  /* signed */
    int tail;  /* signed */
};

#define CIRC_CNT(head,tail,size) (((head) - (tail)) & ((size)-1))
```

`CIRC_CNT` 用 `& (size - 1)` 取代 `% size` 來計算環形緩衝區中的元素數量。`head` 和 `tail` 宣告為 `int`（signed），若 `head - tail` 出現負值，bitmask 會產生一個大正數而非正確的負差值。在正常使用中，呼叫者需確保 `head >= tail`（或理解 wrap around 是刻意設計），但這依賴使用者遵守隱含的前提條件。

相對地，核心中許多較新的實作已改用 unsigned 型別避免此問題，例如 BPF hash table（[`kernel/bpf/hashtab.c`](https://github.com/torvalds/linux/blob/v6.19/kernel/bpf/hashtab.c)）使用 `u32 hash` 搭配 `hash & (htab->n_buckets - 1)`。

### Sign extension 僅在跨圓環搬移時才有意義

- [ ] 為何 sign extension 僅在「跨圓環」搬移時才有意義？從數學觀點解釋，要一般化相關證明

〈[解讀計算機編碼](https://hackmd.io/@sysprog/binary-representation)〉「處理器的指令集和數值編碼」一節提出關鍵觀察：

> 在電腦架構中，實際上每種資料都表示一個圓環，假設系統中有 8 位元、16 位元、32 位元等 3 種資料處理模型的話，系統中就存在三個圓環，只有將一個資料從一個圓環放到另一個圓環的指令才會考慮符號⋯⋯同一圓環上進行的運算根本不用考慮符號，因此也就不用提供兩套指令。

以下從數學上解釋為何同一圓環不需區分 signed/unsigned，而跨圓環搬移必須區分，並將教材的 8 → 16 位元範例一般化為任意 $m$ → $n$ 位元。

#### 同一圓環上的運算不需區分符號

$k$ 位元資料構成的圓環是 $\mathbb{Z}/2^k\mathbb{Z}$，有 $2^k$ 個元素 $\{0, 1, \ldots, 2^k - 1\}$。Signed 和 unsigned 只是同一個圓環上兩種不同的**標記方式**：

- Unsigned：選代表元 $\{0, 1, \ldots, 2^k - 1\}$
- Signed（二補數）：選代表元 $\{-2^{k-1}, \ldots, -1, 0, 1, \ldots, 2^{k-1} - 1\}$

兩者都是 $\mathbb{Z}/2^k\mathbb{Z}$ 的完全剩餘系（complete residue system）——每個等價類恰好選一個代表元。對任意 $a, b \in \mathbb{Z}/2^k\mathbb{Z}$：

$$(a + b) \bmod 2^k$$

不論 $a, b$ 被解讀為 signed 或 unsigned，模除之後得到的位元模式（bit pattern）完全相同。原因是 signed 的代表元 $x - 2^k$（負數）和 unsigned 的代表元 $x$ 屬於同一等價類：$x - 2^k \equiv x \pmod{2^k}$。

因此，加法器不需要知道運算元是 signed 還是 unsigned——硬體只做 $\bmod\ 2^k$ 的加法，結果的位元模式相同。減法和乘法同理：$a - b \equiv a + (2^k - b) \pmod{2^k}$、$(a \times b) \bmod 2^k$ 的結果也與解讀方式無關。這就是為什麼 x86 的 `ADD`、`SUB`、`IMUL` 指令不需區分 signed/unsigned——同一圓環上，一套指令即可。

唯一需要區分的同圓環運算是**除法和比較**：除法結果取決於數學值而非位元模式（$(-7) \div 2 = -3$ vs $249 \div 2 = 124$ 在 8-bit 下），因此 x86 分別提供 `DIV`/`IDIV` 和 `JA`（unsigned above）/`JG`（signed greater）。

#### 跨圓環搬移必須區分符號：一般化證明

現在考慮將資料從較小的圓環 $\mathbb{Z}/2^m\mathbb{Z}$ 搬到較大的圓環 $\mathbb{Z}/2^n\mathbb{Z}$（$m < n$）。我們需要一個映射 $\phi$ 使得目標圓環中的元素**保留原始數值**。

定義二補數解讀函式。對 $k$ 位元的位元模式 $x \in \{0, \ldots, 2^k - 1\}$，其 signed 值為：

$$T_k(x) = x - 2^k \cdot x_{k-1}$$

其中 $x_{k-1}$ 是最高位元（sign bit）。當 $x_{k-1} = 0$ 時 $T_k(x) = x$，當 $x_{k-1} = 1$ 時 $T_k(x) = x - 2^k$。

**目標：** 給定 $x \in \{0, \ldots, 2^m - 1\}$，找到 $y \in \{0, \ldots, 2^n - 1\}$ 使得 $T_n(y) = T_m(x)$。

**情況 1：$x_{m-1} = 0$（非負值）**

$T_m(x) = x$，且 $0 \leq x < 2^{m-1} < 2^{n-1}$，因此 $x$ 在 $n$ 位元中同樣是非負值。

取 $y = x$，則 $T_n(y) = y = x = T_m(x)$。✓

位元操作：高位補零（zero extension），$y = \underbrace{0\ldots0}_{n-m}\ x_{m-1}\ x_{m-2}\ \ldots\ x_0$。

**情況 2：$x_{m-1} = 1$（負值）**

$T_m(x) = x - 2^m < 0$。需要 $T_n(y) = x - 2^m$。

因為目標值是負數，$y$ 在 $n$ 位元中也必須是負數（$y_{n-1} = 1$），所以 $T_n(y) = y - 2^n$。

$$y - 2^n = x - 2^m \implies y = x + 2^n - 2^m$$

驗證 $y$ 的範圍：$2^{m-1} \leq x < 2^m$，因此 $2^n - 2^m + 2^{m-1} \leq y < 2^n$，確實落在 $\{0, \ldots, 2^n - 1\}$ 內。✓

位元操作：$2^n - 2^m = \sum_{i=m}^{n-1} 2^i$，其二進位表示是第 $m$ 到第 $n-1$ 位元全為 1。由於 $x < 2^m$，加上 $2^n - 2^m$ 不會影響低 $m$ 位元：

$$y = \underbrace{1\ldots1}_{n-m}\ x_{m-1}\ x_{m-2}\ \ldots\ x_0$$

因為 $x_{m-1} = 1$，高位全部是 sign bit 的複製。這就是 sign extension。

**合併兩種情況：** 無論 $x_{m-1}$ 是 0 或 1，正確的 $n$ 位元表示都是將 $x_{m-1}$ 複製到高位的 $n - m$ 個位元——即 sign extension。$\blacksquare$

**對比 unsigned 的情況：** 若 $x$ 的 unsigned 值就是 $x$ 本身，則 $y = x$（zero extension）。當 $x_{m-1} = 1$ 時，sign extension 得到 $x + 2^n - 2^m$，zero extension 得到 $x$——兩者在目標圓環中是**不同的元素**。例如 $m = 8, n = 16$ 時，位元模式 `11111111`：

- Unsigned 值 $255$：zero extension → `00000000 11111111` = $255$
- Signed 值 $-1$：sign extension → `11111111 11111111` = $65535$（unsigned），$T_{16}(65535) = -1$

在 $\mathbb{Z}/2^{16}\mathbb{Z}$ 圓環上，$255$ 和 $65535$ 位於不同位置。這就是為什麼「跨圓環搬移」必須知道符號——同一個位元模式在源圓環和目標圓環中對應不同位置，正確的對應取決於解讀方式。

#### 為何同圓環不需要、跨圓環才需要：直觀理解

同一圓環上，$x$ 和 $x - 2^k$ 是「同一個點」——它們是同一等價類的兩個名字，運算結果自然一致。跨圓環時，源圓環的一個點必須映射到目標圓環的某個點，而 signed 和 unsigned 的映射規則不同——前者保留數學值（可能是負數），後者保留位元模式。兩種映射把同一個源點送到目標圓環的不同位置，因此硬體必須提供兩套指令（`movsx` vs `movzx`）。

### Unsigned overflow 是 defined behavior，signed overflow 是 undefined behavior

- [ ] 為何 Linux 核心中 unsigned overflow 是 defined behavior，但 signed overflow 是 undefined behavior？

這是 C 語言標準（C99）的規定，不是 Linux 核心自己定義的。核心用 C 寫，自然受此約束。

#### C99 的規定

**Unsigned overflow——defined：** C99 §6.2.5 第 9 段：

> A computation involving unsigned operands can never overflow, because a result that cannot be represented by the resulting unsigned integer type is reduced modulo the number that is one greater than the largest value that can be represented by the resulting type.

Unsigned 運算的結果自動取 $\bmod\ (max + 1)$，即 $\bmod\ 2^k$。規格書甚至說「can never overflow」——不是「溢位後 wrapping」，而是「定義上就不存在溢位」，因為結果被數學地定義為模除。

**Signed overflow——undefined：** C99 §3.4.3 將 undefined behavior 定義為「本標準不施加任何要求」的行為，第 3 段直接舉例：

> An example of undefined behavior is the behavior on integer overflow.

此處的 "integer overflow" 指的就是 signed integer overflow（因為 unsigned 已被 §6.2.5 定義為不會溢位）。

#### 為什麼標準要這樣區分

**Unsigned 只有一種表示法。** $k$ 位元 unsigned 就是純二進位，所有平台都一樣。溢位後的位元模式只有一種可能（高位被截斷），等價於 $\bmod\ 2^k$，標準可以安全地將此行為寫死。

**Signed 有三種合法表示法。** C99 §6.2.6.2 規定 signed 整數可以是以下任一種（implementation-defined）：

- Sign and magnitude（符號加絕對值）
- Two's complement（二補數）
- Ones' complement（一補數）

三種表示法的溢位行為不同。以 8-bit 的 `127 + 1` 為例：

| 表示法 | `01111111 + 1` 的位元結果 | 解讀 |
|--------|--------------------------|------|
| Two's complement | `10000000` | $-128$ |
| Ones' complement | `10000000` | $-127$ |
| Sign and magnitude | `10000000` | $-0$ |

同樣的位元操作在三種表示法下產生不同的數學值。若標準硬性規定 signed overflow 必須 wrap（像 unsigned 一樣），就必須指定 wrap 到哪個值——但這會偏袒某種表示法，使其他表示法的硬體無法符合標準，或必須插入額外指令來模擬。C 標準選擇留為 undefined behavior，讓各平台按自身硬體的自然行為處理。

#### Undefined behavior 帶來的編譯器最佳化

將 signed overflow 定為 UB 還有一個效果：編譯器可以**假設 signed overflow 永遠不會發生**，從而進行更激進的最佳化。例如對 signed `int x`：

- 編譯器可假設 `x + 1 > x` 永遠為真（因為若不為真，就是 overflow，而 overflow 是 UB，標準不管 UB 的結果）
- 迴圈 `for (int i = 0; i < n; i++)` 可假設 `i` 不會 wrap 回負數，允許向量化等最佳化

這對一般應用程式有效能好處，但對核心程式可能造成危險——編譯器可能刪掉看似「不可能到達」的溢位檢查。

#### 核心如何因應

Linux 核心在頂層 [`Makefile`](https://github.com/torvalds/linux/blob/v6.19/Makefile#L1090) 加入：

```makefile
# disable invalid "can't wrap" optimizations for signed / pointers
KBUILD_CFLAGS	+= -fno-strict-overflow
```

這告訴 GCC：不要假設 signed overflow 不會發生，禁止基於此假設的最佳化。核心選擇犧牲部分最佳化空間，換取可預測的行為——因為核心中有些程式碼刻意依賴 signed 的 wrapping 語意（例如序號比較）。

不過，`-fno-strict-overflow` 只是阻止編譯器利用 UB 做最佳化，並不改變 C 標準的定義。Signed overflow 在語言層級仍然是 UB，核心只是透過編譯器旗標將實際行為固定為 two's complement wrapping。

### 為何核心大量使用 unsigned long 作為時間計數器

- [ ] 為何 Linux 核心大量使用 unsigned long 作為時間計數器？

承接上題：unsigned overflow 是 C99 定義的 $\bmod\ 2^k$ wrapping，而時間計數器必然會溢位——選擇 unsigned long 正是利用這個保證。

#### `jiffies`：核心的基本時間單位

核心的全域時間計數器 `jiffies` 宣告於 [`include/linux/jiffies.h`](https://github.com/torvalds/linux/blob/v6.19/include/linux/jiffies.h#L86)：

```c
extern unsigned long volatile __cacheline_aligned_in_smp jiffies;
```

每次 timer interrupt 觸發，`jiffies` 加 1。頻率由 `HZ` 決定，[`kernel/Kconfig.hz`](https://github.com/torvalds/linux/blob/v6.19/kernel/Kconfig.hz) 提供 100、250、300、1000 四種選項，預設為 250。在 32-bit 系統上 `unsigned long` 是 32 位元，以預設 `HZ=250` 計算，$2^{32} / 250 / 86400 \approx 198.8$ 天就會 wrap 回 0——溢位是必然發生的事。

若用 signed long，溢位就是 undefined behavior（前述「unsigned 與 signed overflow」），編譯器可能產生非預期的程式碼。用 unsigned long，溢位是標準保證的 $\bmod\ 2^k$，行為完全可預測。

#### `time_after()` 巨集：unsigned 減法 + signed 轉型

核心比較兩個時間點的先後時，不能直接用 `a > b`——因為 wrap 之後數值上 `a < b`，但時間上 `a` 其實在 `b` 之後。[`include/linux/jiffies.h`](https://github.com/torvalds/linux/blob/v6.19/include/linux/jiffies.h#L130) 的解法：

```c
#define time_after(a,b)		\
	(typecheck(unsigned long, a) && \
	 typecheck(unsigned long, b) && \
	 ((long)((b) - (a)) < 0))
```

運作原理：

1. `(b) - (a)`：兩個 unsigned long 相減，結果是 unsigned，由 C99 保證 $\bmod\ 2^k$
2. `(long)(...)`：轉型為 signed long，利用二補數的 sign bit 判斷方向
3. `< 0`：若結果為負，代表 `a` 在 `b` 之後（即使 `a` 的數值因 wrap 而小於 `b`）

以 32-bit 為例，假設 `b = 0xFFFFFFF0`，`a = 0x00000010`（`a` 已 wrap）：

- `(unsigned)(b - a) = 0xFFFFFFE0`
- `(long)0xFFFFFFE0 = -32`（負數）→ `time_after(a, b)` 為真

這個技巧能正確處理 wrap，前提是兩個時間點的差距不超過 $2^{k-1}$（即 `LONG_MAX`）。

#### 刻意提早 wrap 來抓 bug

[`include/linux/jiffies.h`](https://github.com/torvalds/linux/blob/v6.19/include/linux/jiffies.h#L317) 中：

```c
/*
 * Have the 32-bit jiffies value wrap 5 minutes after boot
 * so jiffies wrap bugs show up earlier.
 */
#define INITIAL_JIFFIES ((unsigned long)(unsigned int) (-300*HZ))
```

注意巨集的雙重轉型：先轉 `(unsigned int)`（32-bit）得到 `0xFFFEDB08`，再轉 `(unsigned long)`。在 32-bit 系統上 `unsigned long` 就是 32 位元，初始值離 $2^{32}$ 只剩 75000 ticks = 300 秒，5 分鐘後就會 wrap。在 64-bit 系統上，`(unsigned int)` 到 `(unsigned long)` 是 zero extension，初始值變成 `0x00000000FFFEDB08`，離 $2^{64}$ 還有 $1.84 \times 10^{19}$ 個 tick——不會 wrap。

這是刻意的設計：64-bit 系統的 `unsigned long` 本身就不會 wrap（23 億年），不需要測試；32-bit 系統才有約 199 天 wrap 的問題，`INITIAL_JIFFIES` 讓它在開機 5 分鐘後就提早觸發，若有驅動程式錯誤地用 `>` 而非 `time_after()` 比較時間，在開發階段就會暴露 bug。

#### 為什麼是 unsigned long 而非 u32 或 u64

- **`unsigned long`** 隨架構變化：32-bit 系統是 32 位元，64-bit 系統是 64 位元。這使得 `jiffies` 在 64-bit 系統上的 wrap 週期變成 $2^{64} / 250 / 86400 / 365.25 \approx 2.3 \times 10^{9}$ 年——實際上不會溢位
- 核心另外維護 `jiffies_64`（`u64`），在 32-bit 系統上需透過 [`get_jiffies_64()`](https://github.com/torvalds/linux/blob/v6.19/include/linux/jiffies.h#L89) 搭配 sequence lock 才能不可再分地讀取，開銷較大
- 大多數驅動程式只需短時間的計時（毫秒到秒級），32-bit 的 `unsigned long` 足夠且存取成本最低，因此成為預設選擇

### 質數模數構成有限域，與 AES 中 $GF(2^8)$ 的關聯

- [ ] 若模數改為質數 $p$，是否可構成有限域？這和 AES 中 $GF(2^8)$ 有何關聯？

#### 群、環、域的層次關係

先釐清代數結構的層次。群只要求**一個運算**滿足四個性質，域則要求**兩個運算**各自形成群，再用分配律連接：

| 結構 | 要求 |
|------|------|
| 群（group） | 一個運算形成群（封閉、結合、單位元素、反元素） |
| 環（ring） | 加法是阿貝爾群 + 乘法有封閉性和結合律 + 分配律 |
| **域（field）** | 加法是阿貝爾群 + **非零元素的乘法也是阿貝爾群** + 分配律 |

域 = 環 + 乘法反元素。有限域（finite field）就是元素數量有限的域，又稱 Galois field（GF），以數學家 Évariste Galois 命名。

#### $\mathbb{Z}/p\mathbb{Z}$ 是有限域

前述「固定位元與模運算」和「允許溢位與封閉性」討論的 $\mathbb{Z}/2^k\mathbb{Z}$ 在加法下是阿貝爾群（加法 $\bmod\ 2^k$），但乘法不是——因為存在零因子（例如 $2 \times 2^{k-1} \equiv 0 \pmod{2^k}$），不是所有非零元素都有乘法反元素。

若模數改為質數 $p$，元素是 $\{0, 1, \ldots, p-1\}$，加法和乘法都是 $\bmod\ p$。$\mathbb{Z}/p\mathbb{Z}$ 的非零元素 $\{1, 2, \ldots, p-1\}$ 在乘法下構成群（前述「費馬小定理與模反元素」已證明——Latin square 性質保證每個元素都有反元素）。加上加法群的性質，$\mathbb{Z}/p\mathbb{Z}$ 滿足域的全部要求：

| 性質 | $\mathbb{Z}/2^k\mathbb{Z}$（$k > 1$） | $\mathbb{Z}/p\mathbb{Z}$（$p$ 為質數） |
|------|--------------------------------------|--------------------------------------|
| $(\mathbb{Z}, +)$ 阿貝爾群 | ✓（$\bmod\ 2^k$） | ✓（$\bmod\ p$） |
| $(\mathbb{Z} \setminus \{0\}, \times)$ 阿貝爾群 | ✗（有零因子） | ✓（費馬小定理保證反元素存在） |
| 分配律 | ✓ | ✓ |
| **是否為域** | **否（環）** | **是（有限域）** |

$2^k$（$k > 1$）永遠不是質數，這正是 $\mathbb{Z}/2^k\mathbb{Z}$ 無法成為域的根本原因。

關鍵差異在於**零因子**：$p$ 為質數時，$ab \equiv 0 \pmod{p}$ 蘊含 $a \equiv 0$ 或 $b \equiv 0$（因為 $p$ 不能被分解為更小的因子）。$2^k$ 不是質數（$k > 1$ 時），所以 $\mathbb{Z}/2^k\mathbb{Z}$ 有零因子，無法成為域。

#### $GF(2^8)$ 不是 $\mathbb{Z}/256\mathbb{Z}$

[AES](https://zh.wikipedia.org/wiki/%E9%AB%98%E7%BA%A7%E5%8A%A0%E5%AF%86%E6%A0%87%E5%87%86) 加密以 byte（8 位元，256 個值）為單位處理資料。AES 的 S-box 需要對每個 byte 計算**乘法反元素**——這要求 256 個元素構成一個**域**。但 $2^8 = 256$ 不是質數，所以 $\mathbb{Z}/256\mathbb{Z}$ 不是域，不能直接用。

有限域理論保證：對任何質數 $p$ 和正整數 $n$，都存在恰好有 $p^n$ 個元素的有限域 $GF(p^n)$。因此存在一個有 $2^8 = 256$ 個元素的有限域 $GF(2^8)$，但構造方式不是用整數模除，而是**多項式模除**。

$GF(2^8)$ 的構造：
- 基礎域：$GF(2) = \{0, 1\}$，即 $\mathbb{Z}/2\mathbb{Z}$（模 2 算術，加法就是 XOR）
- 元素：$GF(2)$ 上次數小於 8 的多項式，如 $x^7 + x^3 + 1$，共 $2^8 = 256$ 個
- 加法：多項式係數逐項 XOR（因為 $GF(2)$ 中 $1 + 1 = 0$）
- 乘法：多項式相乘後模一個**不可約多項式**（irreducible polynomial），使結果保持在次數 < 8 的範圍內

AES 規範（[FIPS 197](https://csrc.nist.gov/pubs/fips/197/final) §4.2）選用的不可約多項式是 $m(x) = x^8 + x^4 + x^3 + x + 1$（十六進位 `0x11b`，需要 9 個位元表示）。「不可約」意味著它在 $GF(2)$ 上無法分解為更低次多項式的乘積——類似於質數在整數中不可分解。這確保了模除後的乘法具有封閉性且每個非零元素都有反元素。不管金鑰長度是 128、192 或 256 位元，所有 AES 變體內部的 byte 運算都在同一個 $GF(2^8)$ 上進行，使用同一個不可約多項式；金鑰長度影響的是輪數（10/12/14 輪）和金鑰排程，不影響域的選擇。

對比：

| | $\mathbb{Z}/p\mathbb{Z}$ | $GF(p^n)$ |
|---|---|---|
| 元素 | 整數 $\{0, \ldots, p-1\}$ | $GF(p)$ 上的多項式 |
| 加法 | 整數加法 $\bmod p$ | 多項式係數逐項 $\bmod p$ |
| 乘法 | 整數乘法 $\bmod p$ | 多項式乘法 $\bmod m(x)$ |
| 保證封閉的機制 | $p$ 是質數 | $m(x)$ 是不可約多項式 |

兩者的共通結構是：用一個「不可分解的東西」做模除，消除零因子，使非零元素在乘法下形成群。

#### AES 為何選擇 $GF(2^8)$

AES 處理的資料單位是 byte（8 位元），恰好是 $GF(2^8)$ 的 256 個元素。$GF(2^8)$ 的兩個運算特性使其適合硬體實作：

- **加法是 XOR**：不需進位，單一時脈週期完成
- **乘以 $x$（即 `0x02`）是左移一位，溢位時 XOR `0x1b`**

Linux 核心的 AES 實作（[`lib/crypto/aes.c`](https://github.com/torvalds/linux/blob/v6.19/lib/crypto/aes.c#L92)）中，`mul_by_x` 正是這個操作：

```c
static u32 mul_by_x(u32 w)
{
    u32 x = w & 0x7f7f7f7f;
    u32 y = w & 0x80808080;

    /* multiply by polynomial 'x' (0b10) in GF(2^8) */
    return (x << 1) ^ (y >> 7) * 0x1b;
}
```

`0x7f7f7f7f` 遮罩取出每個 byte 的低 7 位元（可安全左移），`0x80808080` 取出最高位元。若最高位元為 1，左移後會超出 8 位元（等同多項式乘以 $x$ 後次數 $\geq 8$），需 XOR `0x1b`（即模 $m(x)$ 的約化）。注意程式中用的是 `0x1b` 而非完整多項式的 `0x11b`——因為溢位時 $x^8$ 項一定為 1（否則不會溢位），XOR 時 $x^8$ 與 $x^8$ 互消，只需處理低 8 位元 $x^4 + x^3 + x + 1$ = `0x1b`。這個函式同時對 4 個 byte 平行運算，用於 MixColumns 的矩陣乘法（[第 111 行](https://github.com/torvalds/linux/blob/v6.19/lib/crypto/aes.c#L111)）。

S-box 則是 $GF(2^8)$ 乘法反元素加上仿射轉換的查表實作（[第 16 行](https://github.com/torvalds/linux/blob/v6.19/lib/crypto/aes.c#L16)），核心使用預先計算好的 256-byte 查找表，不在 runtime 計算反元素。

### 二補數構成環但非域，對除法的限制

- [ ] 二補數構成環但非域，這對除法有何限制？找出 Linux 核心原始程式碼對應的案例

#### 直接回答：環非域 → 乘法沒有反元素 → 除法不封閉

前述「質數模數與有限域」已證明 $\mathbb{Z}/2^k\mathbb{Z}$ 是交換環但非域——因為存在零因子（例如 $2 \times 2^{k-1} \equiv 0$），不是所有非零元素都有乘法反元素。環的乘法只需要封閉性、結合律和單位元素（么半群），不要求反元素；域則進一步要求非零元素在乘法下構成阿貝爾群（含反元素）。

推論鏈：

> **成環但非域** → 乘法沒有通用的反元素 → 除法 $a \div b = a \times b^{-1}$ 無法對所有非零 $b$ 定義 → **除法不封閉**

除法不封閉的意思是：$a \div b$ 的結果可能落在 $\{0, 1, \ldots, 2^k - 1\}$ 之外（例如 $7 \div 2 = 3.5$，不是整數）。加法和乘法的結果會透過溢位（$\bmod 2^k$）自動落回環內，但除法沒有這種天然機制——核心必須用軟體策略把結果映射回去。

#### 核心原始碼中的對策

列舉 [`include/linux/math.h`](https://github.com/torvalds/linux/blob/v6.19/include/linux/math.h)、[`include/linux/math64.h`](https://github.com/torvalds/linux/blob/v6.19/include/linux/math64.h)、[`lib/math/div64.c`](https://github.com/torvalds/linux/blob/v6.19/lib/math/div64.c) 中的除法相關 API：

| 核心 API | 對應的限制 | 策略 |
|----------|-----------|------|
| [`mult_frac`](https://github.com/torvalds/linux/blob/v6.19/include/linux/math.h#L135)、[`DIV_ROUND_UP`](https://github.com/torvalds/linux/blob/v6.19/include/uapi/linux/const.h#L51)、[`DIV_ROUND_CLOSEST`](https://github.com/torvalds/linux/blob/v6.19/include/linux/math.h#L92) | 截斷遺失資訊 | 拆商餘組合；補償截斷方向 |
| [`div_s64_rem`](https://github.com/torvalds/linux/blob/v6.19/lib/math/div64.c#L67)、[`div64_s64`](https://github.com/torvalds/linux/blob/v6.19/lib/math/div64.c#L161) | 有號溢位（$\texttt{INT\_MIN} \div (-1)$） | 轉無號再處理符號 |
| [`__div64_const32`](https://github.com/torvalds/linux/blob/v6.19/include/asm-generic/div64.h#L65) | 反元素不存在 | 倒數近似 + 偏置補償 |
| [`do_div`](https://github.com/torvalds/linux/blob/v6.19/include/asm-generic/div64.h#L45)、[`__div64_32`](https://github.com/torvalds/linux/blob/v6.19/lib/math/div64.c#L32) | 32-bit 無 64-bit 除法 | 軟體長除法 |
| [`__iter_div_u64_rem`](https://github.com/torvalds/linux/blob/v6.19/include/vdso/math64.h#L6)、[`DIV_ROUND_UP_POW2`](https://github.com/torvalds/linux/blob/v6.19/include/linux/math.h#L46) | 效能 | 迭代減法；位元遮罩替代 |

### 二補數是否本質上是種 algebraic encoding

- [ ] 二補數的設計是否本質上是種 algebraic encoding？

從題目脈絡——前面幾題已建立群、環、域的概念——推測老師要問的是：二補數是否為一種**保持代數結構的編碼**，即整數到位元模式的對應，是否保持了加法和乘法的運算關係。要回答這個問題，需要先釐清三個層次的定義。

#### 第一層：什麼是代數結構

代數結構（algebraic structure）是一個集合 $S$ 配上若干運算，且運算滿足特定公理。前面幾題已經用到的三種代數結構：

| 結構 | 定義 | 範例 |
|------|------|------|
| 群（group） | $(S, \circ)$：封閉性、結合律、單位元素、反元素 | $(\mathbb{Z}/2^k\mathbb{Z}, +)$（前述「固定位元與模運算」） |
| 環（ring） | $(S, +, \times)$：加法是阿貝爾群 + 乘法有封閉性和結合律 + 分配律 | $(\mathbb{Z}/2^k\mathbb{Z}, +, \times)$（前述「環非域與除法限制」） |
| 域（field） | 環 + 非零元素的乘法也是阿貝爾群 | $\mathbb{Z}/p\mathbb{Z}$、$GF(2^8)$（前述「質數模數與有限域」） |

代數結構的核心是：**集合 + 運算 + 公理**，三者缺一不可。同一個集合配上不同運算或不同公理，就是不同的代數結構。

#### 第二層：什麼是「保持結構」的映射

兩個代數結構之間的映射，若能保持運算關係，稱為**同態**（homomorphism）。

對環同態 $\varphi: (R_1, +, \times) \to (R_2, +, \times)$，要求：

$$\varphi(a + b) = \varphi(a) + \varphi(b), \quad \varphi(a \times b) = \varphi(a) \times \varphi(b)$$

也就是「先運算再映射」和「先映射再運算」的結果相同。

以整數除以 8 為例：$\ldots, -8, 0, 8, 16, \ldots$ 除以 8 餘數都是 0，歸為同一等價類 $[0]$。整數 $\mathbb{Z}$ 被分成 8 個等價類 $[0], [1], \ldots, [7]$，這些等價類本身構成一個新的環——商環 $\mathbb{Z}/8\mathbb{Z}$。記號中的 $/$ 就是「除以」的隱喻：把大的結構除以子結構，得到更小的結構。

#### 第三層：二補數是環同態的直接結果

現在可以回答核心問題了。證明鏈如下：

**步驟 1：** $(\mathbb{Z}, +, \times)$ 是交換環（整數在加法和乘法下的基本性質）。

**步驟 2：** 定義自然投影 $\pi: \mathbb{Z} \to \mathbb{Z}/2^k\mathbb{Z}$，$\pi(a) = a \bmod 2^k$。

**步驟 3：** $\pi$ 是環同態——保持加法和乘法：

$$\pi(a + b) = (a + b) \bmod 2^k = (\pi(a) + \pi(b)) \bmod 2^k$$
$$\pi(a \times b) = (a \times b) \bmod 2^k = (\pi(a) \times \pi(b)) \bmod 2^k$$

這是模運算的基本性質，成立的原因是 $2^k\mathbb{Z}$（所有 $2^k$ 的倍數）構成 $\mathbb{Z}$ 的理想。

**步驟 4：** 商環 $\mathbb{Z}/2^k\mathbb{Z}$ 有 $2^k$ 個等價類。每個等價類可選擇不同的代表元素：

| 等價類 | unsigned 代表元 | signed（二補數）代表元 |
|--------|----------------|---------------------|
| $[0]$ | $0$ | $0$ |
| $[1]$ | $1$ | $1$ |
| $\vdots$ | $\vdots$ | $\vdots$ |
| $[2^{k-1}]$ | $2^{k-1}$ | $-2^{k-1}$ |
| $\vdots$ | $\vdots$ | $\vdots$ |
| $[2^k - 1]$ | $2^k - 1$ | $-1$ |

兩組代表元是同一商環的**不同完全剩餘系**（complete residue system）。Unsigned 選 $\{0, \ldots, 2^k - 1\}$，signed 選 $\{-2^{k-1}, \ldots, 2^{k-1} - 1\}$。

**關鍵：** 代表元的選擇不影響商環的代數結構。對任意 $a, b$ 及其代表元 $a', b'$（可能是 unsigned 的也可能是 signed 的），只要 $[a'] = [a]$ 且 $[b'] = [b]$，就有：

$$[a'] + [b'] = [a] + [b] = [a + b]$$

這就是為什麼同一套加法器電路對 unsigned 和 signed 都正確——電路操作的是等價類（位元模式），不在乎選了哪個代表元。

#### 結論：二補數是 algebraic encoding

二補數不是任意選擇的位元對應方式，而是以下數學結構的直接結果：

1. 硬體的 $k$ 位元限制自然產生商環 $\mathbb{Z}/2^k\mathbb{Z}$（前述「固定位元與模運算」）
2. 自然投影 $\pi$ 是環同態，保證運算在商環中定義良好（前述「允許溢位與封閉性」）
3. 二補數只是在這個商環中選擇了包含負數的代表元——代數結構完全不變

因此，二補數本質上確實是一種 algebraic encoding：它是環同態下的商結構，選擇不同的等價類代表元來表示有號整數。設計者不是在做位元技巧，而是（有意或無意地）遵循了代數結構的自然映射。前面幾節的討論——群性質（「固定位元與模運算」「允許溢位與封閉性」）、反元素存在性（「費馬小定理與模反元素」）、同圓環不需區分符號（「跨圓環搬移與 sign extension」）——全部都是這個代數結構的不同面向。

### $\mathbb{Z}/2^{64}$ 上的系統時間安全極限

- [ ] 若將系統時間設計於 $\mathbb{Z}/2^{64}$ 上，其安全極限為何？

安全極限取決於**時間單位的精度**——同樣是 64 位元，選擇不同的計時單位，wrap 週期差距極大：

| 時間單位 | unsigned $2^{64}$ | signed $2^{63}$ |
|---------|-------------------|-----------------|
| 秒（`time64_t`） | $5.85 \times 10^{11}$ 年 | $2.92 \times 10^{11}$ 年 |
| jiffies（HZ=250） | $2.34 \times 10^{9}$ 年 | $1.17 \times 10^{9}$ 年 |
| 微秒（μs） | $5.85 \times 10^{5}$ 年 | $2.92 \times 10^{5}$ 年 |
| **奈秒（ns，`ktime_t`）** | **585 年** | **292 年** |

以秒為單位，$2^{63}$ 秒約 2920 億年，遠超[宇宙年齡](https://zh.wikipedia.org/wiki/%E5%AE%87%E5%AE%99%E5%B9%B4%E9%BD%A1)（約 138 億年），實務上不會溢位。但核心的高精度計時器 `ktime_t` 是 **signed 64-bit 奈秒**（`s64`），安全極限只有約 **292 年**——這不是天文數字，而是工程上必須處理的限制。

#### 核心如何定義這些型別

`ktime_t` 的定義在 [`include/linux/types.h`](https://github.com/torvalds/linux/blob/v6.19/include/linux/types.h#L127)：

```c
typedef s64	ktime_t;
```

`time64_t` 和相關常數定義在 [`include/linux/time64.h`](https://github.com/torvalds/linux/blob/v6.19/include/linux/time64.h#L8)：

```c
typedef __s64 time64_t;
typedef __u64 timeu64_t;

#define KTIME_MAX    ((s64)~((u64)1 << 63))
#define KTIME_SEC_MAX    (KTIME_MAX / NSEC_PER_SEC)
```

`KTIME_MAX` 就是 $2^{63} - 1$（`s64` 的最大值），`KTIME_SEC_MAX` 則是 $\lfloor (2^{63} - 1) / 10^9 \rfloor$ 秒——約 292 年又 3 個月。

#### 核心對 292 年極限的因應措施

核心開發者清楚意識到這個限制。[`include/linux/time64.h`](https://github.com/torvalds/linux/blob/v6.19/include/linux/time64.h#L35) 第 35–44 行：

```c
/*
 * Limits for settimeofday():
 *
 * To prevent setting the time close to the wraparound point time setting
 * is limited so a reasonable uptime can be accomodated. Uptime of 30 years
 * should be really sufficient, which means the cutoff is 2232. At that
 * point the cutoff is just a small part of the larger problem.
 */
#define TIME_UPTIME_SEC_MAX    (30LL * 365 * 24 *3600)
#define TIME_SETTOD_SEC_MAX    (KTIME_SEC_MAX - TIME_UPTIME_SEC_MAX)
```

核心限制 `settimeofday()` 可設定的最大時間為 `KTIME_SEC_MAX - 30 年`，確保即使系統連續運行 30 年，`ktime_t` 也不會 wrap。換算下來，`settimeofday()` 的上限大約是西元 **2232 年**。註解中 "At that point the cutoff is just a small part of the larger problem" 意味著：到了 2232 年，如果人類還在用同一套時間系統，292 年的 `ktime_t` 極限只是眾多需要解決的問題之一。

#### 不同時間型別的設計取捨

核心中的時間型別各有不同的精度與範圍取捨：

| 型別 | 定義 | 單位 | 範圍 | 用途 |
|------|------|------|------|------|
| `ktime_t` | `s64` | 奈秒 | ±292 年 | 高精度計時器（hrtimer） |
| `time64_t` | `__s64` | 秒 | ±2920 億年 | 檔案時間戳、wall clock |
| `jiffies` | `unsigned long` | 1/HZ 秒 | 23 億年（64-bit） | 排程器、粗略計時 |

這體現了 $\mathbb{Z}/2^{64}$ 上的根本取捨：**精度越高，範圍越窄**。奈秒精度讓 `ktime_t` 適合微秒級的計時需求（排程、中斷處理），但代價是 292 年的範圍限制。秒精度的 `time64_t` 範圍幾乎無限，但無法處理亞秒級計時。`jiffies` 則介於兩者之間，用 unsigned 避免 signed overflow 的 UB 問題（前述「unsigned 與 signed overflow」）。

### 證明 $(\mathbb{Z}/2^k\mathbb{Z}, +)$ 為阿貝爾群，但 $(\mathbb{Z}/2^k\mathbb{Z}, \times)$ 非群

- [ ] 證明 $(\mathbb{Z}/2^k\mathbb{Z}, +)$ 為阿貝爾群，但 $(\mathbb{Z}/2^k\mathbb{Z}, \times)$ 非群

#### $(\mathbb{Z}/2^k\mathbb{Z}, +)$ 是阿貝爾群

令 $S = \{0, 1, \ldots, 2^k - 1\}$，加法定義為 $(a + b) \bmod 2^k$。逐一驗證五個性質：

1. **封閉性：** $a, b \in S \implies 0 \leq a + b < 2^{k+1}$，取 $\bmod\ 2^k$ 後結果落在 $\{0, \ldots, 2^k - 1\} = S$。這就是前述「允許溢位與封閉性」討論的：硬體捨棄高位元 = $\bmod\ 2^k$，結果自動落回集合。

2. **結合律：** $((a + b) + c) \bmod 2^k = (a + (b + c)) \bmod 2^k$。成立因為整數加法滿足結合律，而 $\bmod$ 運算與加法的順序無關。

3. **單位元素：** $0 \in S$，且 $(a + 0) \bmod 2^k = a$ 對所有 $a \in S$ 成立。

4. **反元素：** 對任意 $a \in S$，取 $-a \equiv (2^k - a) \bmod 2^k$。當 $a = 0$ 時反元素是 $0$；當 $a \neq 0$ 時 $2^k - a \in \{1, \ldots, 2^k - 1\} \subset S$，且 $(a + (2^k - a)) \bmod 2^k = 2^k \bmod 2^k = 0$。每個元素都有反元素。

5. **交換律：** $(a + b) \bmod 2^k = (b + a) \bmod 2^k$。成立因為整數加法滿足交換律。

五個性質全部滿足，$(\mathbb{Z}/2^k\mathbb{Z}, +)$ 是阿貝爾群。$\blacksquare$

#### $(\mathbb{Z}/2^k\mathbb{Z}, \times)$ 非群

只需找到一個性質不成立即可。以下證明**反元素不存在**。

取 $k > 1$（$k = 1$ 時 $S = \{0, 1\}$，$0$ 沒有反元素，同樣非群）。考慮元素 $2$ 和 $2^{k-1}$，兩者都是 $S$ 的非零元素，但：

$$2 \times 2^{k-1} = 2^k \equiv 0 \pmod{2^k}$$

兩個非零元素相乘得到零——$2$ 和 $2^{k-1}$ 是**零因子**（zero divisor）。

零因子不可能有乘法反元素：假設 $2^{-1}$ 存在，對 $2 \times 2^{k-1} \equiv 0$ 兩邊乘以 $2^{-1}$，得到 $2^{k-1} \equiv 0$，但 $2^{k-1} \neq 0$（因為 $k > 1$ 時 $0 < 2^{k-1} < 2^k$），矛盾。因此 $2$ 沒有乘法反元素。

反元素性質不成立，$(\mathbb{Z}/2^k\mathbb{Z}, \times)$ 不是群。$\blacksquare$

## 探討〈你所不知道的 C 語言：指標篇〉

### 回傳區域變數位址的未定義行為

- [ ] 根據 C99 對 object 的定義，請分析以下程式是否具有未定義行為，並說明理由 `int *foo(void) { int x = 10; return &x; }`

`foo` 中的 `x` 沒有 linkage，也沒有 `static` 修飾，依 C99 §6.2.4¶4：

> An object whose identifier is declared with no linkage and without the storage-class specifier `static` has *automatic storage duration*.

automatic storage duration 的 lifetime 規則定義於 §6.2.4¶5：

> its lifetime extends from entry into the block with which it is associated until execution of that block ends in any way.

`foo` 執行 `return &x;` 後，函式的 block 結束，`x` 的 lifetime 隨之結束。而 §6.2.4¶2 明確指出：

> If an object is referred to outside of its lifetime, the behavior is undefined.

因此 caller 拿到的指標指向一個 lifetime 已結束的 object。對該指標做 dereference，就是在 lifetime 外存取 object——未定義行為。

#### 1. 若改成 `static int x = 10;` 結果是否改變？

改變。加上 `static` 後，`x` 的 storage duration 從 automatic 變為 static。§6.2.4¶3：

> An object whose identifier is declared with ... the storage-class specifier `static` has *static storage duration*. Its lifetime is the entire execution of the program.

`x` 的 lifetime 涵蓋整個程式執行期間，`foo` 回傳後該 object 仍然存在。caller dereference 回傳的指標，存取的是一個仍在 lifetime 內的 object，行為完全合法。

#### 2. 請用 storage duration 與 lifetime 兩個術語解釋差異

兩個版本的 `foo` 差異在於 `x` 的 **storage duration** 不同，導致 **lifetime** 不同，最終決定回傳指標的有效性。

未加 `static` 的 `x` 具有 automatic storage duration（§6.2.4¶4），其 lifetime 在所屬 block 結束時結束（§6.2.4¶5）。`foo` return 後 `x` 的 lifetime 已結束，caller 持有的指標指向一個不再存活的 object，dereference 是未定義行為。

加上 `static` 後，`x` 具有 static storage duration（§6.2.4¶3），其 lifetime 是整個程式執行期。`foo` return 後 `x` 仍然存活，caller dereference 回傳的指標完全合法。

一個關鍵字 `static` 改變了 storage duration 的分類，進而改變了 lifetime 的範圍，使得相同的 `return &x;` 從未定義行為變為合法操作。

#### 3. 為何規格明確指出指標值變為 indeterminate？

> The value of a pointer becomes indeterminate when the object it points to reaches the end of its lifetime.

§6.2.4¶2 已經規定「在 lifetime 外存取 object 是 UB」，但規格額外加上這句話，是因為它針對的不是 dereference，而是**指標本身的值**。一個指向 lifetime 已結束之 object 的指標，其值是 indeterminate——即使不 dereference，僅僅比較或複製該指標，行為也是未定義的。

規格之所以特別強調這一點，是因為實務上 stack 記憶體可能尚未被覆寫，指標「看起來」仍然有效。但 C 語言的語意模型中，lifetime 結束就是結束，與底層記憶體是否已被回收無關。編譯器在最佳化時可以依據此規則，假設程式不會持有指向已結束 lifetime 之 object 的指標，進而產生出乎預期的結果。

### `sizeof` 對 array 與 pointer 的差異

- [ ] 以下程式的輸出為何？ `int a[3] = {1,2,3}; int *p = a; printf("%zu %zu\n", sizeof(a), sizeof(p));`

輸出為 `12 8`（以 `gcc -std=c99 -Wall` 編譯執行驗證）。`sizeof(a)` 量的是陣列本身的大小（3 × 4 = 12 bytes），`sizeof(p)` 量的是指標變數本身的大小（x86-64 上為 8 bytes）。

#### 1. 為何 array 在 expression 中會 decay 成 pointer，但在 `sizeof` 中不會？

C99 §6.3.2.1¶3：

> Except when it is the operand of the **`sizeof`** operator or the unary **`&`** operator, or is a string literal used to initialize an array, an expression that has type "array of *type*" is converted to an expression with type "pointer to *type*" that points to the initial element of the array object and is not an lvalue.

規格以「Except」明確列出三個例外：`sizeof`、`&`、以及用來初始化陣列的字串常量。在這三個情境中，array 保留原本的型態不做轉換。因此 `sizeof(a)` 中的 `a` 維持 `int [3]` 型態，算出 3 × `sizeof(int)` = 12；而 `int *p = a;` 中的 `a` 不在例外清單內，decay 為 `int *`，`sizeof(p)` 量的是指標本身 = 8。

#### 2. 為何 `&a` 與 `a` 的值相同但型態不同？

`a` 在一般 expression 中 decay 為 `int *`（§6.3.2.1¶3），指向陣列的第一個元素。

`&a` 中的 `a` 是 `&` 運算子的運算元——同樣被 §6.3.2.1¶3 的 "Except" 排除，不做 decay，保留 `int [3]` 型態。再依 §6.5.3.2¶3：

> The unary `&` operator yields the address of its operand. If the operand has type "*type*", the result has type "pointer to *type*".

運算元型態為 `int [3]`，所以 `&a` 的型態是 `int (*)[3]`（指向整個陣列的指標）。

兩者的數值相同（都是陣列起始位址），但型態不同，差異體現在 pointer arithmetic 上：

- `a + 1`（`int *`）→ 前進 `sizeof(int)` = 4 bytes，指向 `a[1]`
- `&a + 1`（`int (*)[3]`）→ 前進 `sizeof(int [3])` = 12 bytes，指向整個陣列之後

#### 3. 用 Graphviz 繪製記憶體示意圖說明以上
![sizeof_memory](https://hackmd.io/_uploads/SyRvcLEK-g.svg)

### Pointer arithmetic 與 strict aliasing

- [ ] 分析： `double x[3]; int *p = (int *)&x[0]; printf("%d\n", *(p+1));`

`x[0]` 的位址被強制轉型為 `int *` 並賦值給 `p`。`p+1` 前進 `sizeof(int)` = 4 bytes，指向 `x[0]` 的第 4~7 byte（`double` 佔 8 bytes）。`*(p+1)` 將該記憶體區段的 bit pattern 當作 `int` 印出。

此外 `x[3]` 未初始化，其值為 indeterminate（§6.2.4¶5：automatic storage duration 的 object，初始值是 indeterminate），讀取 indeterminate value 本身即是未定義行為。

即使將 `x[0]` 初始化，這段程式仍有更根本的問題——違反 strict aliasing rule。

#### 1. 為何 pointer arithmetic 的單位取決於 type？

C99 §6.5.6¶8：

> When an expression that has integer type is added to or subtracted from a pointer, the result has the type of the pointer operand. If the pointer operand points to an element of an array object, and the array is large enough, the result points to an element offset from the original element such that the difference of the subscripts of the resulting and original array elements equals the integer expression.

規格定義 pointer arithmetic 以「元素」為單位，而非 byte。`p+1` 是指向下一個元素，元素的大小由指標的型態決定。`int *p` 的 `p+1` 前進 `sizeof(int)` bytes，`double *p` 的 `p+1` 前進 `sizeof(double)` bytes。

§6.5.6¶7 進一步指出：

> a pointer to an object that is not an element of an array behaves the same as a pointer to the first element of an array of length one with the type of the object as its element type.

指標與陣列在 C 語言中緊密相關，pointer arithmetic 的設計就是為了讓指標能自然地走訪陣列元素。

#### 2. 這是否涉及 strict aliasing 問題？

是。C99 §6.5¶7（即 strict aliasing rule）：

> An object shall have its stored value accessed only by an lvalue expression that has one of the following types:
> - a type compatible with the effective type of the object,
> - a qualified version of a type compatible with the effective type of the object,
> - a type that is the signed or unsigned type corresponding to the effective type of the object,
> - a type that is the signed or unsigned type corresponding to a qualified version of the effective type of the object,
> - an aggregate or union type that includes one of the aforementioned types among its members (including, recursively, a member of a subaggregate or contained union), or
> - a character type.

`x[0]` 的 effective type 是 `double`，但 `*(p+1)` 透過 `int` 型態的 lvalue 去存取——`int` 與 `double` 不是 compatible type，也不是對應的 signed/unsigned 版本，更不是 character type。不在允許清單內，因此是**未定義行為**。

規格唯一允許的「萬用」存取型態是 **character type**（`char`、`unsigned char`、`signed char`），這也是 `memcpy` 能合法運作的原因。

#### 3. 在 ARMv5 或 RISC-V 上可能出現什麼錯誤？

除了 strict aliasing 造成的語言層級 UB 外，不同硬體架構對記憶體存取有不同的 alignment 要求，同一段程式在不同架構上可能產生截然不同的行為。

**ARMv5**（參考 [ARM Architecture Reference Manual, DDI 0100](https://developer.arm.com/documentation/ddi0406/cb/Appendixes/ARMv4-and-ARMv5-Differences/Application-level-memory-support/Alignment)）：

ARMv5 不支援 unaligned memory access。LDR 指令對非 word-aligned 位址的行為：

- 硬體將位址強制對齊（`addr AND NOT 3`），讀取該對齊位址的 word，再對資料做 byte rotation（`ROR (addr AND 3) * 8`）——讀到的不是預期的資料，而是被旋轉過的值
- 若啟用 alignment checking（SCTLR 的 A bit），直接觸發 **alignment fault**，程式收到 bus error

ARMv6 之後才對大部分單一 load/store 指令支援 unaligned access（見 [ARM Architecture Reference Manual, DDI 0406, A3.2.1 Unaligned data access](https://developer.arm.com/documentation/ddi0406/c/Application-Level-Architecture/Application-Level-Memory-Model/Alignment-support/Unaligned-data-access)；LDM/STM/LDRD/STRD 在 ARMv6+ 仍要求對齊，見 [Unaligned data access restrictions in ARMv7 and ARMv6](https://developer.arm.com/documentation/ddi0406/c/Application-Level-Architecture/Application-Level-Memory-Model/Alignment-support/Unaligned-data-access-restrictions-in-ARMv7-and-ARMv6)）。

**RISC-V**（參考 [RISC-V Instruction Set Manual, Privileged Architecture](https://riscv.github.io/riscv-isa-manual/snapshot/privileged/)）：

RISC-V 的基本整數指令集（RV32I/RV64I）允許實作自行決定是否支援 misaligned access。Privileged spec 定義了 `mcause` 中的 exception cause：

- **Load address misaligned**（cause 4）
- **Store/AMO address misaligned**（cause 6）

多數嵌入式 RISC-V 核心不支援硬體 unaligned access，會觸發 address-misaligned exception，trap 到軟體模擬處理。

### Call-by-value 與指標的指標

- [ ] 解釋為何以下程式不會改變 main 中的 ptrA： `void func(int *p) { p = &B; }` 並分析 `void func(int **p) { *p = &B; }`

C 語言所有的函式參數傳遞都是 call-by-value。呼叫 `func(ptrA)` 時，傳入的是 `ptrA` 的**值**（即 `A` 的位址），而非 `ptrA` 這個變數本身的位址。`func` 內的 `p` 是一份獨立的複本，`p = &B` 只修改了這份複本，`main` 中的 `ptrA` 完全不受影響。

若要在 `func` 內改變 `ptrA` 的指向，必須傳入 `ptrA` 本身的位址，也就是 `func(&ptrA)`。此時 `p` 的型態是 `int **`，存放的是 `ptrA` 的位址。`*p = &B` 透過 `p` 找到 `ptrA`，將其內容改為 `&B`，成功改變了 `main` 中 `ptrA` 的指向。

#### 1. 用 call-by-value 解釋

C99 §6.5.2.2¶4：

> An argument may be an expression of any object type. In preparing for the call to a function, the arguments are evaluated, and each parameter is assigned the value of the corresponding argument.

每個參數都是「被賦值」——是值的複製，不是變數的別名。因此：

- `func(int *p)` 中 `p` 收到的是 `ptrA` 的值（`&A`）的複本。修改 `p` 只改了複本，`ptrA` 不變
- `func(int **p)` 中 `p` 收到的是 `&ptrA` 的值的複本。但透過 `*p` 可以間接存取 `ptrA` 本身，`*p = &B` 改變了 `ptrA` 的內容

這不是特例——C 語言沒有 call-by-reference，一切都是 call-by-value。「傳指標」只是把位址當作值來傳，本質上仍是複製。

#### 2. 用 Graphviz 繪製 stack frame 示意圖

`void func(int *p) { p = &B; }` — `p = &B` 只改了 func 的區域變數，`ptrA` 不變：

![func_pointer](func_pointer.svg)

`void func(int **p) { *p = &B; }` — `*p = &B` 透過 `p` 找到 `ptrA`，改變其指向：

![func_double_pointer](func_double_pointer.svg)

#### 3. 為何「雙指標」這個說法在語意上不精確？

「雙指標」暗示有**兩個指標**，但 `int **p` 是**一個指標**，其 target type 恰好也是指標型態。`*` 的數量描述的是 indirection 的**深度**，不是指標的**個數**：

- `int *p` — 一個指標，指向 `int`
- `int **p` — 一個指標，指向 `int *`

兩者都是一個指標變數，差別只在所指向的型態。此外「雙指標」也容易與 `double *`（指向 `double` 的指標）混淆。精確的說法應為「指標的指標」（pointer to pointer）。
