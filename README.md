# App 3 scaffold — Interrupts & bottom-half

All analysis/answers contained within this README file. I removed some
context written here originally for space and clarity.

## What you do

1. **Theme rename** — `YOURTHEME` and customize the log messages
2. **Run >= 50 presses, idle** (`WITH_LOAD 0`). Record `latency-max` for both paths.
Sem: max = 2284
Notif: max = 1928

3. **Flip to `WITH_LOAD 1`**, rebuild, and run >= 50 presses again. Re-record both paths. Confirm the four load tasks are live (their heartbeat counters climb).
Sem: max = 2768
Notif: max = 31

4. **Induce a failure** — pick ONE and document the symptom:
   - Remove `portYIELD_FROM_ISR(higher_woken)` → notification is delivered, but the task doesn't run until the next tick
   - Remove `IRAM_ATTR` → first-press latency on cold cache spikes 10-100x
   - Replace `xSemaphoreGiveFromISR` with `xSemaphoreGive` → undefined behavior; system may crash
5. **Defend in README** (see prompts below)

## Engineering analysis (README, graded)

1. **What's in your ISR? What's NOT?** List every line. Defend each (or remove it).
What is not in the ISR: There are no block calls, mutex, direct latency calculations, or printf.
                        According to the module 5 notes, the ISR is mainly for indicating that something needs to happen, not actually doing the work within it.
What is in the ISR:
static void IRAM_ATTR button_isr(void *arg)
{
    int64_t now = esp_timer_get_time(); Defense: This line provides accurate timestamps used for latency calculations.

    Defense: These two lines prevent false button readings when the button is not pressed.
    if (now - last_edge_us < DEBOUNCE_US) return;
    last_edge_us = now;

    Defense: Logic analyzer reading of ISR entry relies on this toggle as a flag.
    gpio_set_level(ISR_PULSE_GPIO, 1);

    Defense: Each of these lines work to update information related directly to the ISR status/execution.
    isr_entry_time_us = now;
    presses_observed++;
    BaseType_t higher_woken = pdFALSE;

    Defense: This indicates sem path without blocking since binary does not rely on counting.
    xSemaphoreGiveFromISR(btn_sem, &higher_woken);

    Defense: This, similar to the sem path, handles the notif path but faster.
    vTaskNotifyGiveFromISR(task_notif_handle, &higher_woken);

    Defense: Important to indicate the ISR is going to return.
    gpio_set_level(ISR_PULSE_GPIO, 0);

    Defense: Context switching to prevent overflow and latency errors.
    portYIELD_FROM_ISR(higher_woken);
}

2. **Binary semaphore vs direct task notification** — quote your measured latency numbers. Which is faster? Why?
Sem: max = 2284
Notif: max = 1928
The notif is faster than sem since it does not need to hold up for a queue like the sem.

3. **Latency under load** — quote idle (`WITH_LOAD 0`) vs loaded (`WITH_LOAD 1`) numbers. By what factor does latency increase? Use the priority geometry above (Task A at 15 outranks your bottom half at 12) to explain *which* load task is responsible for the worst-case increase, and why B/C/D are not.
Idle:
Sem: max = 2284 us
Notif: max = 1928 us
Loaded:
Sem: max = 2768 us
Notif: max = 31 us
The higher priority task A blocks the bottom half tasks in idle which lengthens the latency. This means the bottom half tasks cannot run until A is completed.
B/C/D are lower priority (below 12) which means they cannot delay the bottom half.

4. **Induced failure** — name the rule you broke, the predicted symptom, the observed symptom, and how they match (or don't).
I chose to test removing portYIELD_FROM_ISR(higher_woken), predicting issues with the immediate reschedule of the ISR. Upon removal, latency increases by a tick count and is evident in the new max value.
Furthermore, I receive stack overflow errors whic hmake sense considering the boost in latency.

## Common pitfalls

- **Calling `printf` inside the ISR.** `printf` takes a UART mutex. Mutex from ISR = undefined behavior. The scaffold puts logging in the BOTTOM-HALF tasks for a reason.
- **Forgetting `IRAM_ATTR`.** The first interrupt after a long quiescent period has to load the ISR from flash. That's ~10s of µs of cache fill on top of your nominal latency. With `IRAM_ATTR`, the ISR is in always-on internal RAM.
- **Debounce too short.** A clean push-button bounces for 1–10 ms typically. Wokwi's simulated button is clean, but if you wire a real button, drop `DEBOUNCE_US` to something like 10000 µs.
- **Editing the load-task bodies.** Under `WITH_LOAD 1` the four tasks are a fixture, not the assignment. You're timing your ISR path, so leave their bodies alone; tune only the `*_ITERS`/`*_N`/`*_LEN` knobs if you want a heavier or lighter load.
- **Both bottom-half tasks racing on `latency_max_*`.** This is fine for the scaffold (32-bit reads are atomic, and the max-update is benign-racy). In production you'd use atomics or a mutex — that's App 6's lesson.

## Setup in Wokwi

Same shape as App 1. In a fresh Wokwi ESP-IDF project (`https://wokwi.com/projects/new/esp32-s3`):

1. Replace `diagram.json`, `wokwi.toml`, and `main/CMakeLists.txt` with this folder's versions. (App 3 has no `sdkconfig.defaults` &mdash; uses IDF defaults.)
2. Place this folder's `main.c` at `main/main.c` (delete Wokwi's `main/src/`), **or** leave `main/src/main.c` and edit `main/CMakeLists.txt` to use `SRCS "src/main.c"` + `INCLUDE_DIRS "src"`.
3. Confirm `wokwi.toml`'s `firmware` / `elf` paths match `app3_interrupts` (the `project(...)` name in `CMakeLists.txt`).
4. Click &#9654;.

Task Table

| Task | Function           | Period (ms) | WCET measured (µs) | WCET + 30% margin (µs) | Deadline | Priority | Core |
|------|--------------------|------------:|-------------------:|----------------------:|---------:|---------:|-----:|
| A    | Tamper Poll        | 10          | 346                | 449.8                 | 10 ms    | 15       | 1    |
| B    | Attestation Update | 20          | 776                | 1008.8                | 20 ms    | 10       | 1    |
| C    | Audit Log          | 50          | 4982               | 6476.6                | 50 ms    | 5        | 1    |
| D    | Counter Sync       | 100         | 10338              | 13439.4               | 100 ms   | 2        | 1    |
| sem  | Semaphore path     |   N/A       |      N/A           |   N/A                 | N/A      | 12       | 1    |
|notif | Notification path  |   N/A       |      N/A           |   N/A                 | N/A      | 12       | 1    |

Concurrency diagram

                            ----------------------------------------------------------------------
                            |Core 1: Sem and notif paths (first idle, after Task A with loading) |
                            |       Task A (blocks with TaskDelayUntil() under loading)          |
Tamper                      |       Task B                                                       | --> ISR return
 Event   --> ISR Starts --> |       Task C                                                       |
Detected                    |       Task D                                                       |
                            ----------------------------------------------------------------------


Citations:
ChatGPT: Assignment Instruction Organization
https://chatgpt.com/share/6a407c5a-50c0-83ea-adea-c18bb209f34f