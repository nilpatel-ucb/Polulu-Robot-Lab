# Lab 5 — The hardware timer

Section 9.2 builds a stopwatch, then a set of software timers, on the one timer peripheral in the RP2040. The counter is a 64-bit microsecond clock. The chip can only move 32 bits in one bus transfer, and it only has four alarm registers, so the program keeps every scheduled event in a sorted list and points one hardware alarm at whichever event comes first. The program is `src/lab5_template.lf`. It runs on the C target with `single-threaded: true` and `keepalive: true`, and it draws on the Pololu OLED through `src/lib/Display.lf`.

The timer registers are in the [RP2040 datasheet](https://pip-assets.raspberrypi.com/categories/814-rp2040/documents/RP-008371-DS-1-rp2040-datasheet.pdf), page 536. The Pico SDK wrappers around the same peripheral are documented under [`hardware_timer`](https://www.raspberrypi.com/documentation/pico-sdk/hardware.html#hardware_timer).

Programs were checked by flashing them onto the Pololu: hold B and press reset to enter BOOTSEL, build with `lfc`, then run `picotool load -x bin/lab5_template.elf`. The onboard LED is `PICO_DEFAULT_LED_PIN`. Startup drives it as an output so the later callbacks have a pin to toggle.

## The counter

The timer is a free-running counter that increments once per microsecond. The tick is derived from the reference clock, so the count is a stopwatch: the value grows by 1000 every millisecond and by 1,000,000 every second. The full count is 64 bits wide. At one count per microsecond that width wraps after about 585,000 years, which means that on the bench the counter only ever increases.

The RP2040 data bus is 32 bits wide, so the processor cannot pick up all 64 bits in one read. The counter is published as two halves.

```
TIMELR     low 32 bits; reading it latches the high half
TIMEHR     high 32 bits, captured by the TIMELR read
TIMERAWL   low 32 bits, raw, with no latch
TIMERAWH   high 32 bits, raw
```

A single read of the low half is coherent, because 32 bits fits in one transfer. A 64-bit read has to take the latched pair: read `TIMELR`, which freezes the high half, then read `TIMEHR`. Reading the two raw registers on their own can straddle a carry out of the low half and return a value that never existed.

The Pico SDK hides that pair of reads.

```
time_us_32()    the low 32 bits, from TIMERAWL
time_us_64()    the latched 64-bit count, in microseconds
```

The low half on its own wraps every 2^32 microseconds, which is about 71.6 minutes. `time_us_32()` is the right call when the interval of interest is much shorter than that. `time_us_64()` is the call that stays monotonic across the wrap. This lab reads the registers itself. The display path uses `timer_hw->timelr`, and the scheduling path uses `timer_hw->timerawl`. Both are the live microsecond count in its low 32 bits.

## Four alarms

Beside the counter there are four 32-bit alarm registers, `ALARM0` through `ALARM3`. Each one is compared with the low 32 bits of the counter. This program uses alarm 0 (`ALARM_NUM` is 0), which leaves the other three unused.

Writing `timer_hw->alarm[n]` does two things: it stores the match value, and it arms that alarm. The alarm fires when the low half of the counter equals the stored value. The hardware then sets the matching bit in `INTR` and disarms that alarm, so the same match does not sit asserted. Three more registers finish the interrupt path.

```
INTE     which alarms are allowed to raise an interrupt
INTR     which alarms have fired; write 1 to a bit to clear it
ARMED    which alarms are still armed; write 1 to a bit to disarm it
```

Alarm 0 has its own line into the processor, `TIMER_IRQ_0`. The peripheral enable in `INTE` and the processor enable from `irq_set_enabled` are both on before that line can enter a handler. Clearing the bit in `INTR` is what lets the line fall again.

A written match that is already behind the counter fires immediately. That rule is what makes one alarm able to walk a list of timers: after one event is handled, the alarm is loaded with the next expiry, and if that expiry has already passed the handler runs again at once.

## One alarm, many timers

Four match registers cannot hold every delay a program wants, so alarm 0 is shared. Every outstanding timer is a node in `linked_list`, sorted by `timer_value`, soonest at the head. The alarm register is always a copy of the head. The sequence for any number of timers is the same:

```
insert the new timer into the sorted list
write the head's expiry into ALARM0
--- the counter runs ---
ALARM0 matches, the handler runs
call that timer's callback and unlink it
if it repeats, add its period and insert the same node again
write the new head into ALARM0, or disarm the alarm if the list is empty
```

`list_insert_sorted`, `list_remove`, and `list_remove_first` are the list operations supplied with the lab. The head is the smallest `timer_value`. After the handler (or a cancel) changes the list, `timer_hw->alarm[0]` is assigned from the new head, so the next interrupt is the next event and nothing else.

## 9.2.2 Reading the counter

`print_hw_timer` is a Lingua Franca timer with a 1 second period. Its reaction is the only place the OLED is given the hardware count. For this checkoff the call to `virtual_timer_init()` in `reaction(startup)` stays commented out, so no alarm is armed and no interrupt handler is installed. The counter is already running when the RP2040 leaves reset; the reaction only samples it.

The reaction reads `timer_hw->timelr`, formats it with `snprintf` into a static buffer as `"%lu us"`, and sends the text out through `Display`. The display inputs are effects of the reaction, `disp.line0` and `disp.line1`, which is the same shape as the `Display` example. Line 0 is the label `TIMER:` and line 1 is the count. The buffer has to be a string because those ports carry strings. The finished file calls `virtual_timer_init()` from startup, because the later sections share that reaction; commenting the call out is only for this checkoff.

The Lingua Franca timer is the metronome for the sample. The number on the screen is the hardware counter, in microseconds, and it advances by about 1,000,000 between frames.

Checkoff: the OLED shows the hardware timer in microseconds and refreshes every second.

## 9.2.3 Arming an alarm

`virtual_timer_init()` turns alarm 0 into an interrupt source. It sets bit 0 of `INTE` with `hw_set_bits`, installs `lab5_timer_irq_handler` with `irq_set_exclusive_handler(TIMER_IRQ_0, ...)`, and enables that interrupt in the processor with `irq_set_enabled`. `reaction(startup)` calls `virtual_timer_init()` from this section on.

The handler's first job is to clear the interrupt. `hw_clear_bits(&timer_hw->intr, 1u << ALARM_NUM)` writes a 1 to bit 0 of `INTR`, which drops the request. The handler then sets `timer_fire`. A second Lingua Franca timer, `check_fire`, runs every 100 ms, sees the flag, clears it, and writes `TIMER FIRED` to `disp.line2`. The display port is an effect of that reaction, so the OLED update happens in the reaction and the handler only records that the alarm matched.

The match time is an absolute count on the running counter. `virtual_timer_start` computes it as `timer_hw->timerawl + microseconds` and writes that into `timer_hw->alarm[ALARM_NUM]`. Ten seconds is 10,000,000 microseconds, so the write is the current low half plus 10,000,000. The counter is left running; the alarm waits for it.

Checkoff: arm a timer ten seconds ahead and show `TIMER FIRED` once the handler has run.

## 9.2.4 A timer is a list node

From here a timer is a `node_t` allocated with `malloc(sizeof(node_t))`. The node holds the absolute expiry in `timer_value`, the function to call in `callback`, and `next` for the list. `virtual_timer_start` fills those in, marks the timer as a one-shot (`repeated` false, `period` 0), and places the node with `list_insert_sorted`. It then copies the head's expiry into the alarm register, so a timer that is sooner than anything already queued becomes the new match.

The value returned to the caller is `(uint32_t) node`. On this processor a pointer is 32 bits, and each `malloc` returns a distinct address, so the pointer is a timer id that `virtual_timer_cancel` can later turn back into a node.

When the alarm fires, the handler takes the head off with `list_remove_first` and calls `node->callback()`. A one-shot is then `free`d. The callback type is `void (*)(void)`, so the call is `node->callback()` with no arguments. `led_start` and `led0_toggle` are callbacks of that shape: each one flips the LED pin.

Checkoff: start several one-shot timers in `reaction(startup)` whose callback toggles the LED. Each one fires at its own expiry, the LED changes, and the node is freed.

## 9.2.5 Repeated timers

A repeated timer keeps two extra fields on the same node. `repeated` is true, and `period` is the interval in microseconds. `virtual_timer_start_repeated` allocates the node, sets `timer_value` to `timerawl + microseconds`, stores the callback and the period, inserts it, and arms the alarm from the head. The id it returns is again the node address.

The counter keeps running. When a repeated timer is the head that just fired, the handler calls the callback and then does `timer_value += period` on that same node. The node is inserted back into the sorted list, so its id stays valid, and the alarm register is loaded from whatever node is now the head. The next expiry is the previous expiry plus the period, so the cadence stays on the original grid: a callback that runs a little late still schedules the next edge one period after the last edge.

Checkoff: `virtual_timer_start_repeated(1000000, led0_toggle)` toggles the LED once a second for as long as the board runs.

## 9.2.6 Cancel

`virtual_timer_cancel` takes the id returned by start and treats it as a `node_t *`. It unlinks that node with `list_remove` and `free`s it. The alarm register is then brought back in line with the list. If a head remains, `timer_hw->alarm[ALARM_NUM]` is set to that head's `timer_value`. If the list is empty, `timer_hw->armed = 1u << ALARM_NUM` disarms alarm 0, so a timer that was deleted cannot still fire.

The list update and the alarm write sit between `irq_set_enabled(TIMER_IRQ_0, false)` and `irq_set_enabled(TIMER_IRQ_0, true)`. The handler walks the same list. With the interrupt held off, a match cannot arrive halfway through the unlink and free a node the handler is also using.

The id has to be remembered across reactions, so `startup` stores it in the reactor state `blink_id`. A Lingua Franca timer `cancel_timer(5 sec)` is the second reaction, and it calls `virtual_timer_cancel(self->blink_id)`. In the file those two reactions are commented, next to the startup that currently runs the 9.2.8 pair; uncommenting them is the cancel checkoff.

Checkoff: start a one-second repeated toggle, cancel that same id five seconds later, and confirm the LED stops toggling.

## 9.2.7 Six timers at once

With more than one timer queued, every path that changes the list leaves the alarm pointing at the new head.

```
virtual_timer_start             insert, then alarm = head
virtual_timer_start_repeated    insert, then alarm = head
lab5_timer_irq_handler          after the callback, alarm = head if the list is not empty
virtual_timer_cancel            alarm = head, or disarm alarm 0 if the list is empty
```

`list_insert_sorted` puts a new node in expiry order, so the head test is a single read of `linked_list->timer_value`. An empty list is the other case: the handler leaves the alarm alone after the last one-shot, because a fired alarm is already disarmed, and cancel writes `ARMED` when it removes the final node.

The six-timer checkoff is the commented block in the second `reaction(startup)`. Three one-shots call `led_on`, at 1,000,000, 3,000,000, and 5,000,000 microseconds. Three call `led_off`, at 2,000,000, 4,000,000, and 6,000,000. They are independent nodes. The alarm tracks whichever of them is soonest, the handler frees each node after its callback, and the LED turns on at 1 s, 3 s, and 5 s and off at 2 s, 4 s, and 6 s.

## 9.2.8 The list and the alarm stay in step

`list_insert_sorted` and `list_remove` rewrite `next` pointers as they walk. The handler does the same walk. If alarm 0 fired in the middle of an insert, the handler could unlink a node the inserter still considers the head, and the expiry written back to the alarm register would be whichever node each side happened to see. The two views of "the soonest timer" would disagree.

`irq_set_enabled(TIMER_IRQ_0, false)` around the list change and the alarm write makes that sequence one piece. The processor remembers an interrupt that arrives while it is disabled. `irq_set_enabled(TIMER_IRQ_0, true)` delivers it once, through `lab5_timer_irq_handler`. The callback runs in that handler. The 100 ms reaction only copies `timer_fire` onto the OLED and clears the flag, so the same expiry is not handled a second time on the Lingua Franca side. The handler also clears `INTR` before it touches the list, which lets the following alarm write raise a new request of its own.

The startup that is built is the 9.2.8 pair:

```
virtual_timer_start_repeated(1000000, led0_toggle);
virtual_timer_start_repeated(2000000, do_nothing);
```

The LED callback is due every 1,000,000 microseconds and `do_nothing` every 2,000,000. Every two seconds the two expiries land on the same count. The handler runs the head, reinserts it one period later, and writes the new head into `ALARM0`. When that new head is already due, the write fires immediately and the second callback runs before the handler returns to the background. The LED keeps toggling once a second through those shared edges.
