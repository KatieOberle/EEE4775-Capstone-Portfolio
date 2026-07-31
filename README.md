Wokwi Link
__________
https://wokwi.com/projects/467986775393981441 

Project Overview
_________________
  This is a general overview of the Application 3 project that my capstone covers. 
First and foremost, I chose this application out of the five assignments since I felt 
that I had learned the most from this particular program.
  The application was meant as a practice for threads and tasks concepts in real-time 
systems. I first modified the skeleton code provided to match each of the four tasks to 
my chosen theme of hardware security. Hardware security was my theme for the semester as 
I felt the most drawn to it in regards to my career. As a computer engineering major, 
hardware security programs in real-time systems are critical for ensuring that there is 
no tampering or misuse of computer technology. A delayed or incorrect indication of 
malicious activity could mean significant damage towards the customer and a loss of trust
from your market.
  To continue, I moved on to customizing the four tasks to match my chosen, theme-aligned 
tasks including the following: tamper poll, attestation update, audit log, and counter sync. 
The tamper poll task acts as a simulation of a tamper detection. Common tamper detection 
methods include changes in voltage demand that are incongruent with the typical demands. For 
academic purposes, we were encouraged to use a simulated effort. The attestation update and 
audit log ensure proper recording of the tamper event flagged. The counter sync ensures proper 
recording and counting of events with respect to the WCET and period. Finally, the semaphore
and notification paths are used as signaling primitives during the process.
  After customization, I carried out the program analyzing the log and recording necessary 
information for engineering analysis, the task table, and concurrency diagram. These 
organizational tools are industry standard and expected of engineers when justifying their 
programs. Therefore, it was an especially important practice highlighted in this application.
  The simulation proved mostly reliable, however, I would remain cautious with online 
simulations versus physical simulations. Our board of choice was an ESP32 using C embedded 
programming with FreeRTOS.

Demo
_______
<iframe width="560" height="315" src="https://www.youtube.com/embed/wH4IpwB1tHM?si=XA4j3iYqw2-DhL9t" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

If embedded video issues persist: https://www.youtube.com/watch?v=wH4IpwB1tHM

Hazard Analysis
_______________
 While I was provided a robust skeleton for this project, there are
certain aspects of the program that remain under consideration
for future improvement.
(1) xSemaphoreGiveFromISR does not count button presses while
    the notification path does save a count. This means that a
    count recorded by the semaphore path might not match the
    notification path.
(2) When the WITH_LOAD variable is set, the tamper poll task runs
    above the bottom tasks. Alongside with the fact that
    portYIELD_FROM_ISR only reschedules tasks, it is not ensured
    that the next task will run next.
(3) Dual-running the semaphore and notification paths is useful
    to cross reference. However, it is preferred to choose one
    path to run for a real-life application.

Original README Including Task Table and Concurrency Diagram
_____________________________________________________________
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


Final Reflection
________________
  In reflection on this project and course, I was pleasantly surprised at just how 
much is involved in ensuring reliable and effective systems in this realm of 
embedded programming. Specifically to this application, tracking multiple, 
competing tasks felt more real-world particularly with the thematic integration.
  Although this was a rewarding and informative experience, there are areas of
improvement with this application. With time constraints, I unfortunately had
to limit the project to what I could effectively deliver without room for
too much experimentation. If I had the opportunity to spend extra time with this
portfolio project, I would implement more tasks. I would love to limit test
between duplicates of tasks and possibly switching tasks to see where
failures mostly occur and try remedying them. Having learned about more real-
time systems concepts such as mutexes and priority inversion, I would love
to incorporate these in the program, as well. Finally, I would love to try using
a physical ESP32 and cross reference with the Wokwi simulation to see where
discrepancies lie.
  There were parts of the project that were more challenging than expected.
In particular, I was tasked with recording the mean WCET values from over
100 iterations of the program. This appears to be a regular mean calculation,
however, the intricate values and volume of data were too large to be 
confidently calculated without human error. Therefore, I handled this
challenge by uploading my log to an AI tool. The tool efficiently and
accurately provided the mean WCET values. This saved time for other,
more important analysis tasks. I believe this use of AI for redundant or
human-error-prone tasks are important for my future career since it will
define employees who can more efficiently complete tasks and progress.
  Finally, the most valuable lesson that I have learned from this project
and all the interesting projects that I had the fortune of participating
was the skill of technical presentation and justification. In my own
experiences interviewing and working, being able to effectively
communicate technical topics and ideas. The practice of not just
learning the material but also being able to defend my decisions
and resolve questions regarding my process have been incredibly
helpful. As an engineer, you need to be able to defend and prove your
results.
  In conclusion, I am incredibly grateful for the opportunity to grow with this
capstone project. I am also very appreciative of the guidance and wisdom
from my professor, Dr. Mike Borowczak, and the teaching assistants
Marlon Garcia Honores and Tayab Uddin Wara. Our class always had access to the
information we needed, and they all facilitated an encouraging intellectual
space for the students.
