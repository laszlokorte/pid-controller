<script>
    import {onMount} from 'svelte'

    const formatter = new Intl.NumberFormat('en-US',{minimumFractionDigits:3, maximumFractionDigits:3})
    const intFormatter = new Intl.NumberFormat('en-US',{minimumFractionDigits:0, maximumFractionDigits:0})
    
    let svg
    $: svgPoint = svg ? svg.createSVGPoint() : null
    
    let selectedObject = 0

    function clamp(min, max, v) {
        return Math.max(min, Math.min(max, v))
    }
    
    const pidUpdate = (s,o,dt) => {
        if(s.target) {
            const errorX = s.target.x - o.x;
            const PX = s.proportionalGain.x * errorX;
            const errorY = s.target.y - o.y;
            const PY = s.proportionalGain.y * errorY;

            const errorRateOfChangeX = s.lastError ? (errorX - s.lastError.x) / dt : 0;
            const errorRateOfChangeY = s.lastError ? (errorY - s.lastError.y) / dt : 0;
            const valueRateOfChangeX = s.lastValue ? (o.x - s.lastValue.x) / dt : 0;
            const valueRateOfChangeY = s.lastValue ? (o.y - s.lastValue.y) / dt : 0;

            const deriveMeasureX = s.dtype == 'error' ? errorRateOfChangeX : -valueRateOfChangeX
            const deriveMeasureY = s.dtype == 'error' ? errorRateOfChangeY : -valueRateOfChangeY

            const DX = s.derivativeGain.x * deriveMeasureX;
            const DY = s.derivativeGain.y * deriveMeasureY;

            const IX = s.integralGain.x * s.integrationStored.x;
            const IY = s.integralGain.y * s.integrationStored.y;

            return {
                ...s,
                lastError: {x: errorX, y: errorY},
                lastValue: {x: o.x, y: o.y},
                integrationStored: {
                    x: clamp(-s.integralSaturation, s.integralSaturation, s.integrationStored.x + errorX * dt),
                    y: clamp(-s.integralSaturation, s.integralSaturation, s.integrationStored.y + errorY * dt),
                },
                fx: PX + DX + IX,
                fy: PY + DY + IY,
            }
        } else {
            return {
                ...s,
                lastError: null,
                lastValue: null,
                fx: 0,
                fy: 0,
            }
        }
    }
    
    const movementUpdate = (s, o) => {
        
        const fx = s.right - s.left
        const fy = s.bottom - s.top
        const norm = Math.sqrt(fx*fx+fy*fy) || 1
        
        return {
            ...s,
            fx: o.power.x*fx/norm,
            fy: o.power.y*fy/norm,
        }
    }
    
    const onKeyDown = (evt) => {
        const mapping = {
            'ArrowUp':'top',
            'ArrowRight':'right',
            'ArrowLeft':'left',
            'ArrowDown':'bottom',
        }
        const dir = mapping[evt.key] || null
        if(dir) {
            evt.preventDefault()
            objects[selectedObject].movement[dir] = 1
        }
    }
    
    const onKeyUp = (evt) => {
        const mapping = {
            'ArrowUp':'top',
            'ArrowRight':'right',
            'ArrowLeft':'left',
            'ArrowDown':'bottom',
        }
        const dir = mapping[evt.key] || null
        if(dir) {
            objects[selectedObject].movement[dir] = 0
        }
    }
    
    function clickSetTarget(evt) {
        if(!svgPoint) {
            return
        }
        
        svgPoint.x = evt.clientX;
        svgPoint.y = evt.clientY;
        const ctm = svg.getScreenCTM().inverse();
        const p =  svgPoint.matrixTransform(ctm);
        
        objects[selectedObject].pid.target = {x:p.x,y:p.y}
    }
    
    function clickResetTarget(evt) {
        objects[selectedObject].pid.target = null
    }
        
    let objects = [
        {
            name: 'Red Ball',
            x:100,y:0,vx:0,vy:0,fx:0,fy:0,mass:2,radius:50,friction:0.1,elasticity:0.4,airFriction:0.1,color:'#f0a',
            power: {x:40,y:60},
            pid: {
                target: null,
                dtype: 'error',
                proportionalGain: {x:1,y:1},
                derivativeGain: {x:1,y:1},
                integralGain: {x:0.3,y:0.3},
                lastError: null,
                lastValue: null,
                integrationStored: {x:0,y:0},
                integralSaturation: 100,
            },
            movement: {
                left:0,
                right:0,
                top:0,
                bottom:0
            }
        },
        {
            name: 'Orange Ball',
            x:-100,y:-100,vx:0,vy:0,fx:0,fy:0,mass:4,radius:40,friction:0.5,elasticity:0.6,airFriction:0.1,color:'#fa0',
            power: {x:40,y:60},
            pid: {
                target: null,
                dtype: 'error',
                proportionalGain: {x:1,y:1},
                derivativeGain: {x:1,y:1},
                integralGain: {x:0.3,y:0.3},
                lastError: null,
                lastValue: null,
                integrationStored: {x:0,y:0},
                integralSaturation: 100,
            },
            movement: {
                left:0,
                right:0,
                top:0,
                bottom:0
            }
        }
    ]
    
    function acceleration(state, time) {
        return {
            x: state.fx/state.mass - (state.airFriction * state.vx),
            y: state.fy/state.mass + (state.touchBottom ? 0 : 9.81) - (state.airFriction * state.vy),
        };
    }
    
    function evaluate({x,y,vx,vy,fx,fy,mass,touchBottom,airFriction}, t, dt, {dx,dy,dvx,dvy} = {dx:0,dy:0,dvx:0,dvy:0}) {
                const state = {
                    x: x + dx*dt,
                    y: y + dy*dt,
                    vx: vx + dvx*dt,
                    vy: vy + dvy*dt,
                    mass,
                    fx, 
                    fy,
                    touchBottom,
                    airFriction
                }
                
                const acc = acceleration(state, t+dt)
        
                return {
                    dx: state.vx, 
                    dy: state.vy, 
                    dvx: acc.x,
                    dvy: acc.y,
                }
    }
    
    onMount(() => {
        let tickId = null
        let prevTime = null
        const nextTick = (cb) => {
            tickId = requestAnimationFrame((time) => {
                const supSteps = 100
                for(let t=0;t<supSteps;t++) {
                    cb(time, 0.1/supSteps)
                } 
                nextTick(cb)
                prevTime = time
            })
        }
        
        nextTick((time, dt) => {
            objects = objects.map((obj) => {
                const pidResult = pidUpdate(obj.pid, obj,dt)
                const movementResult = movementUpdate(obj.movement, obj)
                return {
                    ...obj,
                    pid: pidResult,
                    fx: obj.power.x/10*pidResult.fx + movementResult.fx,
                    fy: obj.power.y/10*pidResult.fy + movementResult.fy,
                }
            }).map((obj) => {
                const a = evaluate( obj, time, 0.0);
                const b = evaluate( obj, time, dt*0.5, a );
                const c = evaluate( obj, time, dt*0.5, b );
                const d = evaluate( obj, time, dt, c );

                const dxdt = 1.0 / 6.0 * 
                    ( a.dx + 2.0 * ( b.dx + c.dx ) + d.dx );
                const dydt = 1.0 / 6.0 * 
                    ( a.dy + 2.0 * ( b.dy + c.dy ) + d.dy );
                
                const dvxdt = 1.0 / 6.0 * 
                    ( a.dvx + 2.0 * ( b.dvx + c.dvx ) + d.dvx );
                const dvydt = 1.0 / 6.0 * 
                    ( a.dvy + 2.0 * ( b.dvy + c.dvy ) + d.dvy );
                        
                
                const speed = Math.sqrt(dxdt*dxdt + dydt*dydt)
                const acc = Math.sqrt(dvxdt*dvxdt + dvydt*dvydt)
                
                if(obj.touchBottom && speed < dt) {
                    return obj
                }
                            
                
                const newX = obj.x + dxdt * dt
                const newY = obj.y + dydt * dt
                            
                const touchBottom = newY+obj.radius >= 500
                const touchTop = newY-obj.radius <= -500
                const touchRight = newX+obj.radius >= 500
                const touchLeft = newX-obj.radius <= -500
                
                const flipY = (touchBottom || touchTop) ? -obj.elasticity : 1
                const flipX = (touchRight || touchLeft) ? -obj.elasticity : 1
                
                const frictionX =  (touchBottom || touchTop) ? (1-obj.friction) : 1
                const frictionY =  (touchRight || touchLeft) ? (1-obj.friction) : 1
        
                
                return {
                    ...obj,
                    x: touchLeft ? -500+obj.radius: (touchRight ? 500-obj.radius : newX),
                    y: touchTop ? -500+obj.radius : (touchBottom ? 500-obj.radius: newY),
                    vx: frictionX * flipX * (obj.vx + dvxdt * dt),
                    vy: frictionY * flipY * (obj.vy + dvydt * dt),
                    touchBottom,
                    touchTop,
                    touchRight,
                    touchLeft,
                }
            })
        })
        
        return () => {
            cancelAnimationFrame(tickId)
        }
    })
</script>

<style>
    :global(body) {
        background-color: #cfefff;
    }

    h1 {
        margin: 0;
    }

    svg {
        margin: 0;
    }

    label {
        white-space: nowrap;
    }

    legend {
        background-color: #333;
        color: #fff;
        display: inline-block;
        padding: 3px 0.6em;
    }

    fieldset {
        margin: 1em 0 0;
        padding: 1em 1em 0;
        border: 1px solid #aaa;
    }

    details {
        margin: 1em 0;
    }

    summary {
        cursor: pointer;
    }

    .active {
        stroke-width: 5;
        stroke: #ffa;
    }
</style>

<svelte:window on:keydown={onKeyDown} on:keyup={onKeyUp} />

<div style="display: grid;grid-template-columns: 1fr 3fr; max-width: 150ch; margin: 1em auto;">
    
    <div style="box-sizing: border-box; width: 100%; background: #fff; padding: 1em">
    
<h1>
    PID-Controller
</h1>

<p>
    Use your arrow keys to move the selected ball. Double click to select a target position you want the ball to move to on its own. Single click to clear the target.
</p>
<p>
    By tuning the balls PID controller parameters you control the way the ball trys to hold its target position.
</p>

<fieldset>
    <legend>
        Object to control
    </legend>
    
    {#each objects as obj, i}
    <label><input type="radio" value="{i}" bind:group={selectedObject} /> {obj.name}</label>
    {/each}
</fieldset>

<fieldset>
    <legend>Power</legend>

    <dl style="margin: 0; display: grid; grid-template-columns: auto auto auto; justify-content: start;">
        <dt><label for="radius">Radius</label></dt>
        <dd>
            <input min="1" max="100" step="0.01" bind:value={objects[selectedObject].radius} type="range" style:accent-color={objects[selectedObject].color} id="radius" />
        </dd>
        <dd>
            <output>{formatter.format(objects[selectedObject].radius)}</output>
        </dd>
        <dt><label for="mass">Mass</label></dt>
        <dd>
            <input min="0.5" max="12" step="0.01" bind:value={objects[selectedObject].mass} type="range" style:accent-color={objects[selectedObject].color} id="mass" />
        </dd>
        <dd>
            <output>{formatter.format(objects[selectedObject].mass)}</output>
        </dd>
        <dt><label for="power.x">Horizontal Power</label></dt>
        <dd>
            <input min="0" max="100" step="0.01" bind:value={objects[selectedObject].power.x} type="range" style:accent-color={objects[selectedObject].color} id="power.x" />
        </dd>
        <dd>
            <output>{formatter.format(objects[selectedObject].power.x)}</output>
        </dd>

        <dt><label for="power.y">Vertical Power</label></dt>
        <dd>
            <input min="0" max="100" step="0.01" bind:value={objects[selectedObject].power.y} type="range" style:accent-color={objects[selectedObject].color} id="power.y" />
        </dd>
        <dd>
            <output>{formatter.format(objects[selectedObject].power.y)}</output>
        </dd>
    </dl>
</fieldset>

<fieldset>
    <legend>
        PID Controller Parameters
    </legend>
    
    <dl style="margin: 0; display: grid; grid-template-columns: auto auto auto; justify-content: start;">
        <dt><label for="proportionalGain.x">Proportional Gain X</label></dt>
        <dd>
            <input min="0" max="1" step="0.01" bind:value={objects[selectedObject].pid.proportionalGain.x} type="range" style:accent-color={objects[selectedObject].color} id="proportionalGain.x" />
        </dd>
        <dd>
            <output>{formatter.format(objects[selectedObject].pid.proportionalGain.x)}</output>
        </dd>
        <dt><label for="proportionalGain.y">Proportional Gain Y</label></dt>
        <dd>
            <input min="0" max="1" step="0.01" bind:value={objects[selectedObject].pid.proportionalGain.y} type="range" style:accent-color={objects[selectedObject].color} id="proportionalGain.y" />
        </dd>
        <dd>
            <output>{formatter.format(objects[selectedObject].pid.proportionalGain.y)}</output>
        </dd>
        <dt><label for="integralGain.x">Integral Gain X</label></dt>
        <dd>
            <input min="0" max="1" step="0.01" bind:value={objects[selectedObject].pid.integralGain.x} type="range" style:accent-color={objects[selectedObject].color} id="integralGain.x" />
        </dd>
        <dd>
            <output>{formatter.format(objects[selectedObject].pid.integralGain.x)}</output>
        </dd>
        <dt><label for="integralGain.y">Integral Gain Y</label></dt>
        <dd>
            <input min="0" max="1" step="0.01" bind:value={objects[selectedObject].pid.integralGain.y} type="range" style:accent-color={objects[selectedObject].color} id="integralGain.y" />
        </dd>
        <dd>
            <output>{formatter.format(objects[selectedObject].pid.integralGain.y)}</output>
        </dd>
        <dt><label for="integralSaturation">Integral Saturation</label></dt>
        <dd>
            <input min="0" max="200" step="1" bind:value={objects[selectedObject].pid.integralSaturation} type="range" style:accent-color={objects[selectedObject].color} id="integralSaturation" />
        </dd>
        <dd>
            <output>{intFormatter.format(objects[selectedObject].pid.integralSaturation)}</output>
        </dd>
        <dt><label for="derivativeGain.x">Derivative Gain X</label></dt>
        <dd>
            <input min="0" max="1" step="0.01" bind:value={objects[selectedObject].pid.derivativeGain.x} type="range" style:accent-color={objects[selectedObject].color} id="derivativeGain.x" />
        </dd>
        <dd>
            <output>{formatter.format(objects[selectedObject].pid.derivativeGain.x)}</output>
        </dd>
        <dt><label for="derivativeGain.y">Derivative Gain Y</label></dt>
        <dd>
            <input min="0" max="1" step="0.01" bind:value={objects[selectedObject].pid.derivativeGain.y} type="range" style:accent-color={objects[selectedObject].color} id="derivativeGain.y" />
        </dd>
        <dd>
            <output>{formatter.format(objects[selectedObject].pid.derivativeGain.y)}</output>
        </dd>
    </dl>
    <details>
        <summary>Summary Internal state</summary>

        <dl style="margin: 0; display: grid; grid-template-columns: auto auto auto; justify-content: start;">
            
        <dt>Last Error</dt>
        <dd style="grid-column: span 2">
            {#if objects[selectedObject].pid.lastError}
        x:{formatter.format(objects[selectedObject].pid.lastError.x)}, y:{formatter.format(objects[selectedObject].pid.lastError.y)}
            {:else}None
            {/if}
        </dd>
        <dt>lastValue</dt>
        <dd style="grid-column: span 2">
            {#if objects[selectedObject].pid.lastValue}
        x:{formatter.format(objects[selectedObject].pid.lastValue.x)}, y:{formatter.format(objects[selectedObject].pid.lastValue.y)}
            {:else}None
            {/if}
        </dd>
        <dt>integrationStored</dt>
        <dd style="grid-column: span 2">
            x:{formatter.format(objects[selectedObject].pid.integrationStored.x)}, y:{formatter.format(objects[selectedObject].pid.integrationStored.y)}
        </dd>
        </dl>
    </details>
</fieldset>
</div>

<svg bind:this={svg} viewBox="-500 -500 1000 1030" style="width:100%; height: auto;" width="500" height="500" on:dblclick={clickSetTarget} on:click={clickResetTarget}>
    <rect x="-500" y="500" width="1000" height="30" fill="green" />
    <rect x="-500" y="-500" width="1000" height="1000" fill="#adf" />
        <circle cx={320} cy={-300} r="{80}" fill="#ffc" opacity="1"  />

    {#each objects as {x,y,vx,vy,fx,fy,mass,radius,color,pid,movement}, i (i)}
    
    {#if movement}
    <path d="M{x},{y}
    v20l{(movement.left-movement.right)*radius*2},-20
    l{-1*(movement.left-movement.right)*radius*2},-20
    z" stroke-width="5" fill="orange" stroke="white" />
    <path d="M{x},{y}
    h20l-20,{(movement.top-movement.bottom)*radius*2}
    l-20,{-1*(movement.top-movement.bottom)*radius*2}
    z" stroke-width="5" fill="orange" stroke="white" />
    {/if}
    
        <circle class:active={selectedObject===i} cx={x} cy={y} r="{radius*1}" fill="{color}" stroke="{color}" stroke-width="2"  />
    
        {#if pid.target}
    <path d="M{pid.target.x},{pid.target.y}v10v-20v10h10h-20" stroke={color} stroke-linecap="round" stroke-linejoin="round" stroke-width="7" />
    <path d="M{pid.target.x},{pid.target.y}v10v-20v10h10h-20" stroke={"white"} stroke-width="3" stroke-linecap="round" stroke-linejoin="round" />
    {/if}
    <path d="M{x},{y}l{vx},{vy}" stroke="blue" />
    <path d="M{x},{y}l{fx},{fy}" stroke="red" />
    
    
    
    {/each}
    

</svg>

</div>