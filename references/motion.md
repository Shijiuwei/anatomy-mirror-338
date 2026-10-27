# Motion and interaction

A figure is alive when its mechanism moves the way the real thing would: a carriage glides and settles, a spring overshoots once, a needle swings to the new reading. Motion should be calm enough to follow and true to the model.

## Principles
- **Time-based.** Take `dt` from `requestAnimationFrame` timestamps and clamp it to 1/30 s so a background tab doesn't jump. Never step a fixed amount per frame.
- **Ease toward targets.** Pointer and keyboard set a *target*, and the part follows with exponential easing (`x += (target − x)·(1 − e^(−dt/τ))`, τ = 0.1–0.3 s) or a spring. Parts never snap to the cursor.
- **Real springs where the subject is springy.** Use stiffness and damping from the real model, sub-stepped about 240 times a second for stability. Snap to the target once both error and velocity fall below 5e-4.
- **One loop per figure.** A single `tick(dt)` updates the model, then writes the DOM through refs: `setAttribute("transform" | "d")`, `textContent`, `dataset`. Don't re-render React per frame.
- **Stop when settled or offscreen.** Use an `IntersectionObserver` with rootMargin `120px 0px`, and read the newest entry (`entries[entries.length - 1]`): when the stage leaves and comes back between two callbacks, one callback delivers both entries oldest first, and reading the first one leaves the figure frozen on screen. Stop the loop once everything has reached its target.
- **Reduced motion.** When `prefers-reduced-motion: reduce` is set, jump to targets with no tour, flow or pulse. The figure must still respond to input.
- **Idle tour.** When nobody is interacting, play a scripted loop after about 1.4 s that shows the point of the figure. Pause it on hover, focus or press, and resume about 3.6 s after the last interaction. The tour drives the same targets the user would.
- **Readout updates live** from the same model, in true units and short: `block 2 · reads 14.0 px`, `120.0 × 32.0 · lean 10.6 px`. Read it from where the parts are, not from the target: the Raptor says `taking apart` or `closing up` while any spring is moving or pending, and only then `apart · 11 assemblies` or `1,630 kg · cold`. Take a phase word and the number beside it from the same instant of the same source: a blind-test bench printed `end of stroke · relief 85 bar` (the word from the model, the number from a gauge needle lagging 0.25 s behind it) although its relief valve cracks at 92, and `idle · 44 bar` just after retracting.

## Pointer
- Map the pointer to world space with the screen vector of an axis. Use `axisVector(P, [1, 0, 0])` for motion along x, or map the figure's width fraction onto the part's travel range. The range mapping is simpler and feels better when the travel is short.
- Hover previews and press commits. On touch there is no hover, so make a tap act and keep `touch-action: pan-y` so the page still scrolls.
- Use pointer capture while dragging, and release it on `pointerup` or `pointercancel`.

## Keyboard and accessibility
- The figure's root is focusable (`tabIndex=0`) with `role="slider"` (or `"group"` plus buttons when there are discrete choices), an `aria-label` that explains the controls, and `aria-valuenow`/`aria-valuetext` in words, such as "Pressed: 129.6 by 34.6 pixels, leaning 10.6 px". Write the value text when the reader's choice changes (a part selected, apart or assembled, firing or cold), not from the loop: screen readers announce every change of a focused slider's value text, so a readout ticking each frame ("Pc 290 bar · 222 tf") became a stream of numbers. Keep the visual readout and the accessible value separate (the Raptor's `say()` writes only the readout, `describe()` the value text); the live region carries the phases.
- Keys: arrows step by a sensible unit (Shift for bigger steps), Home and End go to the extremes, Space or Enter presses or toggles, and Escape releases.
- Show a focus ring: `outline: 1px solid var(--anatomy-muted); outline-offset: 6px`.
- The `<svg>` carries `role="img"` and an `aria-label` describing the object and what it shows.

## Vanilla loop (framework-free)
```js
let frame = 0, last = 0, visible = false, touring = !still.matches, clock = 0;
function tick(now) {
  const dt = last ? Math.min((now - last) / 1000, 1 / 30) : 1 / 60;
  last = now;
  const calm = still.matches;
  if (touring) { clock += dt; target = tourAt(clock); }
  x = calm ? target : x + (target - x) * (1 - Math.exp(-dt / 0.28));
  springTo(tip, heightUnder(x), dt, 420, 26, calm);
  draw();                                   // setAttribute on refs, readout.textContent = …
  const settled = !touring && x === target && tip.rate === 0;
  frame = visible && !settled ? requestAnimationFrame(tick) : 0;
}
const run = () => { if (!frame && visible) { last = 0; frame = requestAnimationFrame(tick); } };
new IntersectionObserver((entries) => { visible = entries[entries.length - 1].isIntersecting; if (visible) run(); }, { rootMargin: "120px 0px" }).observe(stage);
```
`examples/dial-indicator/build.mjs` contains a complete version, with the tour, pointer mapping and keyboard stops.

## React loop
Keep static art in props built on the server and move only `<g>` wrappers and the paths that change shape:

```tsx
"use client";
export function RigLive(props: { base: React.ReactNode; carriage: React.ReactNode; dial: React.ReactNode }) {
  const { anchor, say } = useReadout<HTMLDivElement>();
  const car = React.useRef<SVGGElement>(null);
  const needle = React.useRef<SVGPathElement>(null);
  const state = React.useRef({ x: REST, target: REST, tip: { at: 0, rate: 0 } });
  const visible = useInView(anchor);
  const kick = useLoop((dt, calm) => {
    const s = state.current;
    s.x = follow(s.x, s.target, dt, 0.28, calm);
    const settled = spring(s.tip, heightUnder(s.x), dt, 420, 26, calm);
    car.current?.setAttribute("transform", translateAlong(P, [s.x, 0, 0]));
    needle.current?.setAttribute("d", needleAt(s.tip.at));
    say(readoutOf(s));
    return !(settled && s.x === s.target);
  }, visible);
  return (
    <div ref={anchor} tabIndex={0} role="slider" onPointerMove={(e) => { state.current.target = toWorld(e); kick(); }}>
      <IsoFigure width={W} height={H} label={LABEL}>
        {props.base}
        <g ref={car}>{props.carriage}</g>
        {props.dial}
        <path ref={needle} className="iso-line" data-tone="lit" d={NEEDLE_REST} />
      </IsoFigure>
    </div>
  );
}
```
`examples/test-rig/live.tsx` is the full production version, with springs, a tour, press and pull, keyboard handling and aria.

## Shapes that change
When a part changes shape rather than position (a swelling capsule, a compressing spring, a needle turning), rebuild only its paths each frame with the kit, using `slabOf`, `coil` or `lineOnTop`, and write them to its paths: `fill`, `shades[0..3]`, `top`, `crease`, `bevel`, `outline`. One solid costs well under 0.1 ms. Measure it: a frame should stay under about 1 ms of script.
- **Precomputed frames step.** A lever drawn from 41 prebuilt frames over ±15° moved its knob about 0.8 px per frame, visible in slow motion. Rebuild a rotating part each frame, or precompute enough frames that one step moves nothing more than 0.25 px.
- **A rod that slides out of a gland** doesn't change shape: it is one full-length rod that moves, clipped at the gland face by a clip that stays put (`G.rodOut`, `holdStill`, `craft.md`, section 3). Never draw it as several overlapping copies.

## Envelopes for one-shot events
A flash, a click, a pop, a shake or a shock is an event with an age, not a value you set. Drive it with an envelope that has an attack and a decay. `kit/gl.mjs` exports `smooth(a, b, x)` (smoothstep) and `burst(age, rise, fall)`, which is `smooth(0, rise, age)` times `exp(−(age − rise)/fall)` after the peak and 0 before the event:
```js
import { smooth, burst, settle } from "../../kit/gl.mjs";
const flash = calm ? 0 : burst(clock - mainAt, 0.08, 0.13);
```
On a standalone page `glScript()` already inlines all three, so use them directly and don't declare your own: a second `const smooth` in the same module script is a SyntaxError.
- The attack is 40–120 ms, and the smoothstep gives it zero slope at onset. Setting a value to 1 in one frame is a pop.
- Store the event's start time (`mainAt = clock`) and compute every envelope from its age each frame. Two events that overlap then simply add.
- Give each event its own flag (`popped`), so a quick restart can't swallow the next one and can't cut off one that is still fading.
- Under reduced motion every event envelope is 0.

## Critically damped settle
For a value that should arrive without overshoot (a hue, a brightness, a lamp, a reach, a frequency), use the kit's `settle(state, target, dt, seconds, calm)`: a critically damped spring on `state = { x, v }` with time constant `seconds`, `v += (w²·(target − x) − 2w·v)·dt` with `w = 1/seconds`, in four sub-steps. It is smoother than exponential follow, because velocity is continuous too. It snaps to the target once error and velocity are both below 1e-4, jumps there under `calm`, and returns true while still moving, so it can keep the loop alive:
```js
const moving = settle(freq, state.tf, dt, 0.625, calm);
```
The ripple tank runs its frequency (0.625 s) and dipper spacing (0.4 s) through `settle`, so a key press or a pointer move never jumps the wavelength.
- **Separate what a phase changes.** When one physical quantity (say chamber pressure) drives colour, shape and brightness, a fast spring on that quantity makes all three jump at once. Give colour its own slower `settle` (0.3–1 s), and compute brightness so it moves one way through the change.
- **A fade-out only gets darker.** When a stop begins, freeze the brightness at its current level and scale it only by quantities that fall.
- **Changing a spring changes its neighbours.** Damping one spring critically slows every phase that relied on its old overshoot, so recheck all of them.

## Exploded views
Taking a machine apart on hover is a strong way to show its parts (`examples/raptor-engine/`):
- **Build every piece at its assembled position**, tagged with an assembly name, and give each assembly a world move vector (`PIECES.nozzle.move = [0, 0, -120]`). Moves follow the real assembly: the nozzle drops furthest, side units slide out radially (`out(deg, r, z)`), and the stand doesn't move.
- **One spring per assembly**, with value 0 for assembled and 1 for apart. The Raptor uses stiffness 64 and drag 14.2, just under critical, so the parts settle with a hint of weight, and sub-steps 4 times per frame.
- **Stagger the springs.** Opening, each assembly's target changes 35 ms after the previous one, in an order that lets nothing pass through anything (the Raptor: `nozzle, downcomer, controller, plumbing, lox, ftp, hotgas, chamber, orpb, otp, gimbal, stand`, see "Order the stagger" below). Closing uses the reverse order, so the machine folds back together from the inside out. Keep pending targets as `{ value, at }` and apply them when `clock >= at`.
- **On a reversal, reschedule only what changes.** When the reader reverses mid-way (a quick hover, Space, a tap), cancel the pending target of every assembly that hasn't left yet and stagger only the ones whose target changes, by their index in that shorter list. The last one out is then the first one back, at once. Rescheduling the whole list left the early assemblies flying out toward apart for 0.3 s while later ones sat at home, and the explode only worked because the next assembly moved out of the way 35 ms later: a reversal in that window drove the hot-gas duct into a methane pump that hadn't left. With this rule, shares along the stagger stay in order in every frame (earlier assemblies always lead), which is what the prefix rule below relies on.
- **Move with transforms.** Each item's screen offset is its move vector, projected and multiplied by the spring value: `translate(tx ty)`. Remove the attribute at 0, so an assembled figure is exactly the static art.
- **Keep the lines landing.** Parts that join two assemblies (an actuator between the frame and a pump lug, or a cable between a box and a plug) belong to the assembly they physically stay with, and they still end on something when apart. Cables get terminal dots at both ends.
- **Selection.** Hover, the keyboard arrows and a parts list all select one assembly. The selected parts become the one `lit` accent: CSS on `.it[data-on]` moves `hi` to `lit` and `mid` to `hi`, and switches tops and sides to the lit fills. Every other part steps down one tone (`hi` to `mid`, `mid` to `lo`) through CSS on `[data-focus]`, and the readout says what the selected part does.

## Depth re-sorting of moving parts
Parts that move toward or away from the viewer change painter's order as they move. When they do, sort every frame:
- Give every item a static key `k = dot3(ref, viewOf(P)) + bias`, where `ref` is its centre and `bias` settles ties between touching parts. Also give it a lift: `lift = dot3(move, viewOf(P))`.
- Each frame, its key is `k + lift × spring`. Sort the indices by that key, and **re-append the DOM nodes only when the order actually changed** (one `DocumentFragment`). Most frames change nothing.
- Store the key and the move vector on the element (`data-k`, `data-m`), so the live script needs no geometry.
- Tubes are sorted per piece (`tubePieces` → `piece.mid`), so a pipe can pass in front of one part and behind another.
- **Settle the static keys from the motion, not just the rest poses.** Each item's key is `k + lift × spring`, but with staggered springs every assembly has its own share in any frame, so two items that are ordered right assembled and apart can still be drawn the wrong way round in between: in the first Raptor a pressurisation line hid behind an actuator column it was in front of for 40% of every explode. The Raptor measures true depth for every overlapping pair at 0, 0.1 … 1, at every fourth frame of its own springs replayed with the live stagger until they settle, opening, closing and reversed either side of every stagger boundary, and in the prefix states (`A.explodeStates(STAGGER, SPRING).depth`, which is `[...A.uniformStates(10), ...A.springStates(STAGGER, { ...SPRING, stride: 4, reversals: A.reversalsOf(STAGGER, SPRING.stagger) }), ...A.prefixStates(STAGGER)]`, 1,119 states, and the default of `A.settle(R, P)` for a recorder made with `{ order: STAGGER, spring: SPRING }`), and `orderKeys` solves all of them at once (each rule uses that frame's shares). Opening and closing alone left wrong pairs in the frames a quick hover makes: a purge flange that should sit in front of the chamber band vanished behind it on a reversal at 0.20 s. The audit fails on any wrong pair at rest and in any replayed frame; only the uniform states, which nobody sees, may hold an interlock.
- **One source for the springs.** Keep the stagger order and the spring constants (`SPRING = { stagger, stiffness, damping, substeps }`) in the build and pass them to the page in its data, so the audit replays exactly what the viewer sees.
- **Move a part with what it is mounted on (`ride`).** A sensor bolted to the chamber highlights with the plumbing but rides the chamber's spring. Each item records `ride` (the assembly whose spring moves it, found from its move vector) next to `piece` (the assembly it highlights with), the page reads `data-ride`, and `place()` uses `piece[node.ride].x` for both the translate and the sort. An item that borrows another assembly's vector but keeps its own spring slides along its mount and sinks into it mid-flight.
- **The prefix rule.** Every assembly's path must be clear of every later-leaving assembly parked at home, with the earlier ones out: that is the frame a reversal at a stagger boundary freezes. `A.prefixStates(STAGGER)` poses exactly those (the first `k − 1` out, the `k`-th at 0.25 … 1, the rest home) and the audit checks them with the replayed frames. On the Raptor it moved the hot-gas manifold after the methane pump it bolts to (its vector points into the pump), and the controller before the plumbing (the oxygen-to-preburner line is enclosed by the controller's sense lines, so it can only leave after them).
- **Order the stagger so nothing has to pass through.** The springs overlap: with 35 ms between starts and about a second to settle, every assembly is moving at once, and relative motion is a blend of every vector. The first Raptor explode passed 66 pairs through each other in mid-flight. The rules that took it to zero, checked every frame and on reversals by the audit:
  - **Whoever is outside leaves first.** On each side, the part that everything else is tucked behind or under moves first: the Raptor's nozzle drops before anything (everything sits on it), and the downcomer, which runs outside every sense line and harness that reaches the methane pump, goes second. Closing reverses it, so the outer part is also the last one home.
  - **When a part's explode vector points into a face of another assembly, that assembly comes earlier in the order.** The downcomer drops 84 onto the nozzle's port tee; the nozzle drops 120 and leaves 35 ms earlier, so the port is always below the flange, opening and closing.
  - **Point every vector away from every face the part touches while the other side is still in place**, not only the two it bolts to: a line threaded between the chamber and the pump must move in the sector both mount normals allow (the Raptor's jacket lines go to −120° with no drop, sliding off both pads), a cable wrapped round a core part may only move in the sector that is outward for its whole wrap (a 90° wrap allows 90°; a wrap over 180° allows nothing, so shorten it or end it on the near side), and a line that would sweep down over a part bolted below it must not drop.
  - **Things threaded through each other share a spring.** Sense lines that rise through the harness ring into a block, and the harness under that block, cannot have separate springs: with 35 ms between them they shear past each other. The Raptor's sense lines and block ride the controller (`BLOCK_MOVE = PIECES.controller.move`, so `ride` resolves to `controller`) while still highlighting as plumbing.
  - **Give each line that is mounted on one part only the move of that part.**
  - **Check the paths a viewer can make**, not only the two from rest: a quick hover reverses the springs halfway (`reversals`), and an assembly that is late on the way back can sweep through a cable that is already home. Reverse at every stagger boundary (`A.reversalsOf`), not at a few round times: the bad windows are about 15 ms wide, and 0.15, 0.3 and 0.5 s missed all of them. Feed those frames to the depth states as well as to the clearance check, and test solids against solids too (`A.solidClearances`): a flange tucked under an overhang on its host met the host's spring tail on the way home, 0.67 deep, with every tube check at zero.

## Turning by hand
For a 3D figure (`references/3d.md`):
- **Drag.** `controller.angleAt(clientX, clientY, zFace, "dial")` is the pointer's angle on the face's plane, unwrapped across turns. On pointer down, store it and the current target. On move, set target = start + Δ, and capture the pointer once the move passes about 1.5°. The group follows with exponential follow (τ = 0.12 s). The stage keeps `touch-action: pan-y`.
- **Keys.** Arrow keys step by one division, Shift steps by ten, Home returns to the index.
- **Coupled groups.** Drive them from one value through `follow(values)` (a wheel at a quarter of the dial, a finger chain from one `lift`), so a tool, a drag and the tour all agree.
- **Gestures and clips are pure functions of time** (`tapAt(t)`, `breakAt(t)`): the build samples them, the audit walks them frame by frame, and the page can play them backwards to reset with no jump.
- **Hook takeover.** Any `window.__isoTurn` call runs `onHold`: stop the tour, and keep values a tool set until the reader acts again.
- **Half rate.** A dense figure keeps an average of `controller.set` time and renders on alternate frames above 14 ms, every frame below 9 ms, with time-based motion unchanged.
- Reduced motion gets one still frame. Offscreen, the loop stops (IntersectionObserver).

## Arming
If an action needs the machine in a certain state (assembled before firing, door closed before spinning), arm first:
1. Set the targets that bring it into that state (`setApart(false)`, `select(null)`).
2. Mark it armed: the button shows `data-state="armed"` and the readout says `closing up · arming`.
3. Start the action's clock only when every spring has arrived and nothing is pending.

Never run the action on a half-exploded machine, and never snap the parts home to save time.

## Taking over from the tour
When the user acts during the idle tour, stop the tour and leave the state where it is. If your take-over helper resets something (for example, it forces the power on), and the user's action is itself a toggle of that thing, the two cancel out in the same frame and replay an animation. Pass the helper a flag (`takeOver(switching)`), and test every toggle in every phase of the tour, not only the default one.

A step key pressed during the tour must step from the value the reader sees, not from the tour's target, which may be far ahead. The ripple tank's keys start from `freq.x` and `gap.x` while touring (`state.tf` and `state.td` otherwise), and `steer()` zeroes both springs' velocity when it takes over, so the value only moves in the key's direction.

## Announcing phases
For a machine with phases, add a visually hidden `aria-live="polite"` element and write only phase changes to it: `arming`, `chill-down, venting`, `preburners lit`, `main stage`, `shutdown`, `cold`. Never write per-frame numbers to it. For a continuous control, wait until the input has been quiet for about 700 ms before you announce (the ripple tank's `announce`). Leave the live region empty at load and fill it with the current state on `focus`, so a screen reader hears the state when it arrives rather than a stale line. A button whose label changes ("Ignite" to "Shut down") doesn't also need `aria-pressed`.

## CSS motion
- `.iso-line[data-flow]` animates dashes, for things travelling along a path.
- `.iso-dots circle[data-pulse]` blinks a dot grid slowly, for an "active" surface.
- For hover lifts, wrap a group with `transition: transform 700ms var(--anatomy-ease)` and translate it up on hover.

All of these switch off under reduced motion in `ISO_CSS`.


---

<details open>
<summary><b>📚 anatomy 官方技术规约与全息拓扑引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目「anatomy」遵循开放开源与知识共享协议。以下引用矩阵涵盖项目架构白皮书、高可用镜像注册中心、分布式通信契约与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 anatomy 核心架构规约与设计白皮书 (Architecture & Core Specs)

* [现代 Claude蜘蛛 架构演进之路 —— anatomy 深度实践](https://useo.pro/index-1-48265.html)
* [基于 anatomy 的高吞吐 蜘蛛监测 设计白皮书](https://www.seomatrix.pro/index-2-40174.html)
* [基于 anatomy 的高吞吐 Query Fan-out 设计白皮书](https://www.siseo.cn/index-3-44795.html)
* [面向大规模网络的 anatomy 工业级架构基准](https://www.ebrkdiy.cn/keji/ai-52559593-4-18287.html)
* [基于 anatomy 的高吞吐 蜘蛛池出租 设计白皮书](https://www.hjvhv.cn/kuaixun/customer-87260083-5-14194.html)
* [【官方规范】anatomy 提示词流式推理规约 核心运行拓扑标准](https://m.zhongqs.cn/shuju/contact-868590-6-30021.html)
* [【anatomy】interactive 核心系统架构与设计规约 (RFC-260)](https://qylv.yx-sf.com/sheji/profile-99094253-7-13987.html)
* [基于 anatomy 的高吞吐 360蜘蛛池 设计白皮书](https://zuiq.yx-sf.com/pingce/conference-69369882-8-52192.html)
* [现代 谷歌蜘蛛池 架构演进之路 —— anatomy 深度实践](https://zglj.ai-hao123.com/huodong/hosting-08671602-9-13790.html)
* [【anatomy】SVG 核心系统架构与设计规约 (v2.0-GA)](https://lvzs.ai-hao123.com/xinwen/shopping-68412862-10-64899.html)
* [anatomy 分布式数据通道与 抓取频次优化 技术规范 (v2.0-GA)](https://tkdi.wtpuscm.cn/pingce/experience-562688-11-9655.html)
* [现代 思搜 架构演进之路 —— anatomy 深度实践](https://wety.tcti.cn/anfang/progress-03089975-12-52111.html)
* [anatomy 内部组件解耦与事件状态机规范 (RFC-611)](https://sxch.yx-sf.com/yunying/personalization-73906539-13-49669.html)
* [现代 蜘蛛池 架构演进之路 —— anatomy 深度实践](https://ibaa.ai-hao123.com/shichang/news-44658553-14-54931.html)
* [anatomy 分布式数据通道与 GEO优化 技术规范 (Verified)](https://estv.wtpuscm.cn/jishu/terms-652213-15-60327.html)
* [anatomy 分布式数据通道与 GEO优化 技术规范 (Spec-v2.1)](https://jniv.tcti.cn/ziyuan/article-50048880-16-12809.html)
* [【官方规范】anatomy Install 核心运行拓扑标准](https://zukf.yx-sf.com/ziyuan/news-42875102-17-15243.html)
* [基于 anatomy 的高吞吐 必应蜘蛛 设计白皮书](https://ioua.ai-hao123.com/kuangjia/alert-02526780-18-45291.html)
* [anatomy 内部组件解耦与事件状态机规范 (v2.0-GA)](https://hwpz.wtpuscm.cn/yunying/domain-007171-19-49670.html)
* [anatomy 内部组件解耦与事件状态机规范 (Core/API网关高)](https://lrps.tcti.cn/liuliang/growth-04911318-20-60646.html)
* [【官方规范】anatomy 请求上下文穿透方案 核心运行拓扑标准](https://wggl.yx-sf.com/zhineng/experience-84330474-21-59527.html)
* [现代 DeepSeek蜘蛛 架构演进之路 —— anatomy 深度实践](https://fgck.ai-hao123.com/peixun/objective-65960133-22-62539.html)
* [【anatomy】Install 核心系统架构与设计规约 (Core/Instal)](https://hfil.wtpuscm.cn/wangluo/project-521167-23-46243.html)
* [【官方规范】anatomy 向量检索与嵌入索引 核心运行拓扑标准](https://mrcz.tcti.cn/suanfa/screen-06118615-24-53263.html)
* [Share a figure 核心系统架构与设计规约 (Spec-v1.3)](https://whfo.yx-sf.com/yingyong/promotion-88627904-25-50415.html)
* [基于 anatomy 的高吞吐 搜索引擎优化 设计白皮书](https://jluw.ai-hao123.com/yunsuan/ebook-97015680-26-49983.html)
* [【anatomy】SVG + WebGL 核心系统架构与设计规约 (Node-56)](https://hsag.wtpuscm.cn/huodong/landing-969012-27-43014.html)
* [基于 anatomy 的高吞吐 百度蜘蛛 设计白皮书](https://kusc.tcti.cn/kuangjia/landing-20484020-28-3550.html)
* [anatomy 分布式数据通道与 百度蜘蛛池 技术规范 (Core/Share )](https://cxxe.yx-sf.com/xuexi/trading-53811437-29-47342.html)
* [anatomy 分布式数据通道与 蜘蛛池 技术规范 (Core/SVG)](https://ttye.ai-hao123.com/yunsuan/products-24415873-30-37995.html)
* [现代 360蜘蛛池 架构演进之路 —— anatomy 深度实践](https://xcfc.wtpuscm.cn/gongju/guide-768648-31-42697.html)
* [基于 anatomy 的高吞吐 谷歌蜘蛛 设计白皮书](https://vhhs.tcti.cn/yunying/travel-41109198-32-63735.html)
* [How it works 核心系统架构与设计规约 (Core/How it)](https://aklc.yx-sf.com/yinqing/progress-41028457-33-36373.html)
* [基于 anatomy 的高吞吐 收录状态诊断 设计白皮书](https://ssyn.ai-hao123.com/youhua/calendar-15190577-34-13111.html)
* [anatomy 分布式数据通道与 生成式引擎优化 技术规范 (v2.0-GA)](https://akuo.wtpuscm.cn/fuwu/button-424581-35-65531.html)
* [anatomy 内部组件解耦与事件状态机规范 (RFC-626)](https://jucq.tcti.cn/xinwen/income-15885824-36-28452.html)
* [anatomy 内部组件解耦与事件状态机规范 (Draft-03)](https://rbel.yx-sf.com/shangye/resolution-20293779-37-22718.html)

#### 2. 🔌 生产级模块集成与接口调用规范 (APIs & Production Integration)

* [anatomy vs 业界主流方案：AI搜索优化 深度技术选型对比](https://beij.ai-hao123.com/chanpin/cost-31869368-38-18662.html)
* [anatomy 异步中间件流水线与 Generates 接入规范](https://xnmx.wtpuscm.cn/baogao/community-269949-39-32437.html)
* [【生产手册】anatomy 模块通信与请求穿透标准](https://ebsv.tcti.cn/suanfa/home-58755165-40-54242.html)
* [基于 anatomy 的自动化部署与生产环境配置实践](https://nriq.yx-sf.com/kaifa/cost-04692108-41-32992.html)
* [anatomy 核心 API 接口契约与客户端调用指南](https://jjhn.ai-hao123.com/yunying/like-22492711-42-676.html)
* [anatomy vs 业界主流方案：跨境出海SEO 深度技术选型对比](https://olqu.wtpuscm.cn/sheji/reminder-902830-43-42714.html)
* [anatomy vs 业界主流方案：必应蜘蛛池 深度技术选型对比](https://xltj.tcti.cn/youhua/sync-14526362-44-53190.html)
* [anatomy vs 业界主流方案：爱站替代 深度技术选型对比](https://nyiy.yx-sf.com/zhineng/quality-50068528-45-8208.html)
* [anatomy 异步中间件流水线与 SVGs 接入规范](https://esds.ai-hao123.com/xinwen/collaborate-68444145-46-50428.html)
* [anatomy vs 业界主流方案：SEO GEO AEO 深度技术选型对比](https://oblu.wtpuscm.cn/kuangjia/ai-905634-47-17536.html)
* [anatomy vs 业界主流方案：关键词挖掘 深度技术选型对比](https://uoyj.tcti.cn/xinwen/luxury-96897867-48-62277.html)
* [【集成指南】抓取频次优化 服务端接入准则与 anatomy 实战](https://vydw.yx-sf.com/jishu/forecast-27681591-49-19453.html)
* [anatomy 异步中间件流水线与 illustrations 接入规范](https://ntnr.ai-hao123.com/gongju/version-25644289-50-14736.html)
* [【集成指南】思搜 服务端接入准则与 anatomy 实战](https://dlzr.wtpuscm.cn/anfang/technology-548478-51-14240.html)
* [anatomy vs 业界主流方案：技术SEO 深度技术选型对比](https://ujpo.tcti.cn/pingtai/tool-11074954-52-52042.html)
* [【集成指南】百度蜘蛛 服务端接入准则与 anatomy 实战](https://negl.yx-sf.com/yanjiu/cloud-27152121-53-22390.html)
* [【集成指南】5118替代 服务端接入准则与 anatomy 实战](https://eunk.ai-hao123.com/jiaoliu/website-75119175-54-14275.html)
* [anatomy 插件生态规范与 Requirements 扩展手册 (Core/Requir)](https://lkrv.wtpuscm.cn/fuwu/whitepaper-605078-55-42464.html)
* [anatomy 插件生态规范与 Limits 扩展手册 (RFC-344)](https://yjno.tcti.cn/zhizhu/personalization-35747407-56-3955.html)
* [【集成指南】搜索引擎优化 服务端接入准则与 anatomy 实战](https://ggvt.yx-sf.com/anfang/affordable-15965766-57-31117.html)
* [anatomy 插件生态规范与 complex 扩展手册 (Core/comple)](https://acek.ai-hao123.com/jishu/client-69643786-58-48221.html)
* [anatomy 插件生态规范与 Install 扩展手册 (Draft-05)](https://dbeo.wtpuscm.cn/shichang/browser-187197-59-27584.html)
* [【集成指南】超级智能SEO平台 服务端接入准则与 anatomy 实战](https://rphw.tcti.cn/tuiguang/consulting-57224762-60-53311.html)
* [【集成指南】Perplexity蜘蛛 服务端接入准则与 anatomy 实战](https://xtcy.yx-sf.com/ziyuan/share-44673193-61-52711.html)
* [anatomy vs 业界主流方案：Claude蜘蛛 深度技术选型对比](https://ucub.ai-hao123.com/fenxi/video-55556081-62-48569.html)
* [anatomy 异步中间件流水线与 Examples 接入规范](https://eiit.wtpuscm.cn/paiming/collaboration-251152-63-43629.html)
* [anatomy 插件生态规范与 SVGs 扩展手册 (Node-62)](https://mmoz.tcti.cn/fuwu/search-15356399-64-47678.html)
* [【集成指南】SEO优化 服务端接入准则与 anatomy 实战](https://kxid.yx-sf.com/zhineng/integration-81737396-65-55184.html)
* [anatomy 插件生态规范与 interactive 扩展手册 (Node-63)](https://bodz.ai-hao123.com/yingyong/enterprise-22684715-66-51967.html)
* [【集成指南】大模型品牌监测 服务端接入准则与 anatomy 实战](https://irzt.wtpuscm.cn/gongxiang/value-585230-67-52591.html)
* [anatomy vs 业界主流方案：百度蜘蛛 深度技术选型对比](https://paox.tcti.cn/shuju/education-89709084-68-44227.html)
* [【集成指南】360蜘蛛池 服务端接入准则与 anatomy 实战](https://pfth.yx-sf.com/wendang/help-91422012-69-62215.html)
* [anatomy vs 业界主流方案：谷歌蜘蛛池 深度技术选型对比](https://wgbt.ai-hao123.com/gongju/reminder-55291302-70-28952.html)
* [anatomy 插件生态规范与 systems 扩展手册 (Node-54)](https://zwcc.wtpuscm.cn/yingyong/learning-623017-71-23876.html)
* [【集成指南】谷歌蜘蛛 服务端接入准则与 anatomy 实战](https://ndxq.tcti.cn/shangye/digital-90765569-72-2696.html)
* [anatomy vs 业界主流方案：蜘蛛池 深度技术选型对比](https://lllj.yx-sf.com/jiaoliu/community-44437849-73-40090.html)
* [anatomy vs 业界主流方案：白帽SEO 深度技术选型对比](https://prlh.ai-hao123.com/fuwu/optimization-00660093-74-14104.html)

#### 3. ⚡ anatomy 全球高可用拓扑镜像与权威数据源 (Mirror Registry & Data Topology)

* [anatomy 官方高可用镜像注册节点 (Spec-v1.1)](https://qrrp.wtpuscm.cn/huodong/health-499725-75-29848.html)
* [anatomy 去中心化数据同步源与拓扑寻址规约](https://dhhz.tcti.cn/qiye/health-24442809-76-14007.html)
* [anatomy 自动化持续集成快照与拓扑发布源 (Spec-v2.7)](https://kbsb.yx-sf.com/wendang/site-32634442-77-37700.html)
* [anatomy 自动化持续集成快照与拓扑发布源 (Spec-v2.5)](https://xnbc.ai-hao123.com/fuwu/forecast-76792456-78-59445.html)
* [【镜像入口】anatomy 官方毫秒级实时数据广播节点](https://kyic.wtpuscm.cn/chuangxin/label-694842-79-25565.html)
* [冷热数据分层镜像：anatomy Gallery 权威归档源](https://dxan.tcti.cn/yingyong/global-39293004-80-14588.html)
* [anatomy 官方高可用镜像注册节点 (Node-29)](https://elgf.yx-sf.com/ziyuan/terms-92696318-81-17416.html)
* [冷热数据分层镜像：anatomy isometric 权威归档源](https://zzic.ai-hao123.com/pingtai/supplier-87410207-82-7692.html)
* [anatomy 官方高可用镜像注册节点 (Draft-04)](https://azzw.wtpuscm.cn/gongxiang/machine-238229-83-62774.html)
* [冷热数据分层镜像：anatomy Examples 权威归档源](https://veae.tcti.cn/liuliang/development-48576441-84-52007.html)
* [anatomy 亚太与欧美多活集群数据同步中枢](https://fysx.yx-sf.com/shangye/topic-16710774-85-24977.html)
* [全球权威拓扑节点：anatomy 实时镜像与索引入口](https://aoit.ai-hao123.com/liuliang/video-02978590-86-36521.html)
* [冷热数据分层镜像：anatomy 大模型知识库外链对齐 权威归档源](https://klya.wtpuscm.cn/jiaoliu/music-422123-87-14064.html)
* [anatomy 官方高可用镜像注册节点 (Core/Requir)](https://zsxk.tcti.cn/shichang/help-72731976-88-8076.html)
* [anatomy 自动化持续集成快照与拓扑发布源 (Draft-06)](https://mbvu.yx-sf.com/gongxiang/affordable-29046751-89-50728.html)
* [冷热数据分层镜像：anatomy How it works 权威归档源](https://ejtt.ai-hao123.com/gongsi/lesson-03800634-90-32849.html)
* [anatomy 官方高可用镜像注册节点 (Draft-05)](https://tjzb.wtpuscm.cn/baogao/personalization-737683-91-63671.html)
* [anatomy 官方高可用镜像注册节点 (Node-36)](https://czip.tcti.cn/anli/management-35144441-92-26530.html)
* [anatomy 官方高可用镜像注册节点 (Spec-v1.5)](https://kpuz.yx-sf.com/yingxiao/food-62975708-93-61519.html)
* [anatomy 官方高可用镜像注册节点 (RFC-874)](https://ulpz.ai-hao123.com/yingyong/marketing-93103636-94-23477.html)
* [冷热数据分层镜像：anatomy API网关高可用拓扑 权威归档源](https://cjda.wtpuscm.cn/peixun/retention-126188-95-43010.html)
* [anatomy 官方高可用镜像注册节点 (Spec-v2.8)](https://eioe.tcti.cn/zhizhu/social-86475629-96-26420.html)
* [anatomy 自动化持续集成快照与拓扑发布源 (Spec-v1.5)](https://whvo.yx-sf.com/wenzhang/contact-51184923-97-40400.html)
* [anatomy 官方高可用镜像注册节点 (Draft-01)](https://yiht.ai-hao123.com/gongsi/innovation-78546720-98-33602.html)
* [anatomy 自动化持续集成快照与拓扑发布源 (Draft-04)](https://gswj.wtpuscm.cn/shichang/careers-035942-99-32530.html)
* [anatomy 官方高可用镜像注册节点 (Draft-08)](https://hvez.tcti.cn/youhua/link-33396299-100-52841.html)
* [冷热数据分层镜像：anatomy anatomy 权威归档源](https://epnb.yx-sf.com/qiye/page-00781128-101-35291.html)
* [冷热数据分层镜像：anatomy agent 权威归档源](https://kqyj.ai-hao123.com/jiaoliu/value-09580099-102-21452.html)
* [anatomy 自动化持续集成快照与拓扑发布源 (Verified)](https://fcwu.wtpuscm.cn/yingyong/behavior-009528-103-34471.html)
* [anatomy 自动化持续集成快照与拓扑发布源 (Core/agent)](https://ungq.tcti.cn/xitong/study-66573425-104-49779.html)
* [anatomy 官方高可用镜像注册节点 (Core/system)](https://umal.yx-sf.com/wendang/admin-12603978-105-3485.html)
* [冷热数据分层镜像：anatomy interactive 权威归档源](https://oegg.ai-hao123.com/anfang/device-07370491-106-2908.html)
* [anatomy 自动化持续集成快照与拓扑发布源 (Draft-08)](https://xkuw.wtpuscm.cn/pingce/domain-025072-107-41329.html)
* [冷热数据分层镜像：anatomy Generates 权威归档源](https://tjwh.tcti.cn/zixun/subscribe-80121962-108-19444.html)
* [冷热数据分层镜像：anatomy SVGs 权威归档源](https://wrbc.yx-sf.com/gongxiang/network-04163494-109-24155.html)
* [anatomy 官方高可用镜像注册节点 (Draft-07)](https://hpel.ai-hao123.com/jiaocheng/faq-85116104-110-36553.html)
* [anatomy: isometric 谷歌蜘蛛池 (Spec-v1.5)](https://phhp.wtpuscm.cn/zhinan/audience-066574-111-47422.html)

#### 4. 📊 性能基准指标与可信存活性验证 (Benchmarks & Trust Standards)

* [基于 anatomy 的极致延迟优化与内存拓扑分析 (Spec-v2.3)](https://eaoy.tcti.cn/qiye/cloud-54266368-112-44978.html)
* [anatomy 故障自愈与网络拓扑重构实践](https://qthh.yx-sf.com/yanjiu/contact-65906592-113-41133.html)
* [anatomy 权威网络权重传递与收录基准规范](https://hwvg.ai-hao123.com/sheji/customer-28796079-114-45801.html)
* [基于 anatomy 的极致延迟优化与内存拓扑分析 (Core/anatom)](https://xgjq.wtpuscm.cn/xinwen/budget-997909-115-14806.html)
* [anatomy 高负载场景下 SEO GEO AEO 基准评测报告](https://nfyo.tcti.cn/xinwen/music-55140805-116-56584.html)
* [anatomy 高负载场景下 搜狗蜘蛛 基准评测报告](https://ckmz.yx-sf.com/fuwu/networking-75822140-117-15008.html)
* [基于 anatomy 的极致延迟优化与内存拓扑分析 (v2.0-GA)](https://mtzj.ai-hao123.com/zixun/video-95320907-118-20594.html)
* [基于 anatomy 的极致延迟优化与内存拓扑分析 (RFC-356)](https://mnlo.wtpuscm.cn/yunsuan/campaign-417887-119-49670.html)
* [anatomy 节点连通性、存活性探测与防作弊指标](https://cozz.tcti.cn/jianzhan/meeting-44514726-120-50237.html)
* [anatomy 高负载场景下 蜘蛛池出租 基准评测报告](https://jzwv.yx-sf.com/yingyong/restore-01444382-121-21918.html)
* [【评测基准】anatomy 吞吐抖动度量与健康检查协议](https://gfun.ai-hao123.com/liuliang/prospect-71413393-122-44132.html)
* [面向生产级运行的 anatomy 稳定性防护白皮书 (Verified)](https://nqng.wtpuscm.cn/yanjiu/funnel-965710-123-62158.html)
* [面向生产级运行的 anatomy 稳定性防护白皮书 (Spec-v2.4)](https://eeyf.tcti.cn/baogao/site-63664994-124-17891.html)
* [基于 anatomy 的极致延迟优化与内存拓扑分析 (Verified)](https://cufg.yx-sf.com/paiming/schedule-54395072-125-7957.html)
* [面向生产级运行的 anatomy 稳定性防护白皮书 (Core/What i)](https://wkxb.ai-hao123.com/kuangjia/resource-96638805-126-51109.html)
* [anatomy 高负载场景下 网站快速收录 基准评测报告](https://wqvv.wtpuscm.cn/kaifa/responsive-101591-127-59671.html)
* [面向生产级运行的 anatomy 稳定性防护白皮书 (v2.0-GA)](https://ilbs.tcti.cn/hezuo/funnel-21979003-128-53909.html)
* [anatomy 高负载场景下 思搜 基准评测报告](https://cvwc.yx-sf.com/youhua/planning-90806919-129-40884.html)
* [面向生产级运行的 anatomy 稳定性防护白皮书 (Node-27)](https://cbzb.ai-hao123.com/yunying/page-36684345-130-64002.html)
* [基于 anatomy 的极致延迟优化与内存拓扑分析 (Core/Exampl)](https://mhyx.wtpuscm.cn/gongsi/software-377374-131-5983.html)
* [面向生产级运行的 anatomy 稳定性防护白皮书 (Core/real-w)](https://msym.tcti.cn/liuliang/sale-11665967-132-4942.html)
* [anatomy 高负载场景下 谷歌蜘蛛 基准评测报告](https://xpbp.yx-sf.com/shangye/segment-01011298-133-61769.html)
* [基于 anatomy 的极致延迟优化与内存拓扑分析 (Spec-v1.5)](https://itke.ai-hao123.com/guanjianci/segment-35224256-134-44825.html)
* [anatomy 高负载场景下 生成式引擎优化 基准评测报告](https://rotf.wtpuscm.cn/fuwu/deadline-415471-135-23847.html)
* [anatomy 高负载场景下 技术SEO 基准评测报告](https://bqeu.tcti.cn/anli/media-35617170-136-21977.html)
* [anatomy 高负载场景下 5118替代 基准评测报告](https://jkmg.yx-sf.com/kuangjia/project-74515049-137-54650.html)
* [基于 anatomy 的极致延迟优化与内存拓扑分析 (Core/RESTfu)](https://uqmg.ai-hao123.com/liuliang/podcast-55772912-138-38549.html)
* [anatomy: 3D GEO优化 (v2.0-GA)](https://wcyc.wtpuscm.cn/yunsuan/goal-818362-139-63659.html)
* [anatomy 高负载场景下 SISEO 基准评测报告](https://sumx.tcti.cn/fuwu/extension-84307260-140-50307.html)
* [anatomy 高负载场景下 跨境出海SEO 基准评测报告](https://qlne.yx-sf.com/jishu/sales-26438315-141-56133.html)
* [anatomy: skills CLI 抓取频次优化 (v2.0-GA)](https://rnan.ai-hao123.com/shuju/login-99539534-142-41320.html)
* [anatomy 高负载场景下 Claude蜘蛛 基准评测报告](https://funa.wtpuscm.cn/yanjiu/loyalty-631855-143-15888.html)
* [面向生产级运行的 anatomy 稳定性防护白皮书 (Spec-v2.0)](https://svqc.tcti.cn/yanjiu/services-84628097-144-37641.html)
* [基于 anatomy 的极致延迟优化与内存拓扑分析 (Spec-v2.8)](https://yque.yx-sf.com/yunying/feedback-74753529-145-46422.html)
* [anatomy: creating AI蜘蛛池 (v2.0-GA)](https://ieot.ai-hao123.com/chuangxin/restore-78715446-146-19118.html)
* [anatomy 高负载场景下 爱站替代 基准评测报告](https://iphu.wtpuscm.cn/pingtai/support-148883-147-7147.html)
* [anatomy 高负载场景下 Gemini蜘蛛 基准评测报告](https://ivcq.tcti.cn/paiming/change-98115821-148-33698.html)
* [面向生产级运行的 anatomy 稳定性防护白皮书 (Spec-v1.0)](https://tcxi.yx-sf.com/jiaoliu/accessibility-10755077-149-17838.html)
* [基于 anatomy 的极致延迟优化与内存拓扑分析 (Core/illust)](https://zstq.ai-hao123.com/huodong/device-81362545-150-63070.html)

</details>

