<script>
	import Mesh from './Mesh.svelte'
	import Link from './Link.svelte'
	import durand from 'durand-kerner'
	
	const formatter = new Intl.NumberFormat('en-US', { maximumFractionDigits: 2, minimumFractionDigits: 2 });
	const signedFormatter = new Intl.NumberFormat('en-US', { maximumFractionDigits: 2, minimumFractionDigits: 2, 'signDisplay': 'always' });
	const svgNumber = new Intl.NumberFormat('en-US', { maximumFractionDigits: 2, minimumFractionDigits: 2 });

	function formatComplex(c) {
		const re = formatter.format(c[0])
		const im = signedFormatter.format(c[1])
		return `${re=='-0.00'?'0.00':re}${im=='-0.00'?'+0.00':im}i`
	}

	const sampleRate = 40

	const examples = {
		'Identity': {
			a: [1],
			b: [],
		},
		'Low pass': {
			a: [.25,.25],
			b: [1,-.5]
		},
		'High pass': {
			a: [-2.90, +4.90, -1.90, -0.40, -0.30, -0.20, +0.80],
			b: [ -1.00, -0.68, -0.28],
		},
		'Unstable': {
			a: [.2,-.3,0.4,-0.1],
			b: [-1.1,0.2,0.7],
		},
		'Invalid': {
			a: [0,1],
			b: [],
		},
	}

	function loadExample(name) {
		a = examples[name].a.slice()
		b = examples[name].b.slice()
		selectedPoleZero = null
	}

	let selectedPoleZero = null

	let a = [1]
	const minOrderA = 0
	const maxOrderA = Math.floor(sampleRate / 2)
	
	
	let b = []
	const minOrderB = 0
	const maxOrderB = Math.floor(sampleRate / 2) - 1


	function findRoots(arr) {
		while(arr[0] == 0){
		    arr.shift();            
		}

		const roots = durand(arr);

		return roots.length && roots[0].length ? roots[0].map((_,i) => [roots[0][i], roots[1][i]]) : []
	}

	function compareComplex(a,b) {
		return Math.sign(Math.atan2(...b) - Math.atan2(...a))
	}

	function toPoleZero(a,b) {
		const zeros = findRoots(a.map(v=>v).reverse()).sort(compareComplex)
		const poles = findRoots([-1,...b].map((v)=>-v).reverse()).sort(compareComplex)

		const polynomeOrder = Math.max(zeros.length, poles.length)

		while(zeros.length < polynomeOrder) {
			zeros.push([0,0])
		}

		while(poles.length < polynomeOrder) {
			poles.push([0,0])
		}

		return [poles, zeros]
	}

	$: unsolvable = a[0] == 0 && a.some(v=>v!=0)

	$: [poles, zeros] = toPoleZero(a,b)

	$: poleMagnitudesSquares = poles.map((p) => p[0]*p[0] + p[1]*p[1])

	$: rocRadiusMin = Math.sqrt(Math.max(0, ...poleMagnitudesSquares))
	$: stable = rocRadiusMin < 1


	let inputName = "dirac"
	const inputs = {
		dirac: Array(sampleRate).fill(0).map((v,i) => i==Math.round(sampleRate/2) ? 1 : v),
		sin: Array(sampleRate).fill(0).map((v,i) => Math.sin(2*Math.PI*i/sampleRate)),
		cos: Array(sampleRate).fill(0).map((v,i) => Math.cos(2*Math.PI*i/sampleRate)),
		const: Array(sampleRate).fill(0).map((v,i) => 1),
	}
	
	$: input = inputs[inputName]

	function getOutputAt(input, a, b, i, cache = []) {
		if(i<0) return 0;
		if(cache[i] === undefined) 

		cache[i] = a.reduce((sum, aa, j) => sum + aa*(input[i-j]||0), 0) +
			b.reduce((sum,bb,k) => sum + bb * getOutputAt(input, a, b, i-k-1, cache), 0)

		return cache[i]
	}

	$: output = input.map((_,i) => getOutputAt(input, a, b, i))
	$: outputFormularTime = 'y[n] = ' + ([
			...a
			.map((c,i) => c==0 ? false : i > 0 ? `${signedFormatter.format(c)} ⋅ x[n-${i}]` : `${formatter.format(c)} ⋅ x[n]`)
			.filter((t) => t != false),
			...b
			.map((c,i) => c==0 ? false : i > 0 ? `${signedFormatter.format(c)} ⋅ y[n-${i+1}]` : `${formatter.format(c)} ⋅ y[n-1]`)
			.filter((t) => t != false)
		].join(' ') || 0)

	$: outputFormularFreq = 'Y(z) = ' + [([
			...a
			.map((c,i) => c==0 ? false : i > 0 ? `${signedFormatter.format(c)}⋅z^(-${i}) ⋅ X(z)` : `${formatter.format(c)} ⋅ X(z)`)
			.filter((t) => t != false),
		].join(' ') || 0), ([
			...b
			.map((c,i) => c==0 ? false : `${(i>0?signedFormatter:formatter).format(c)}⋅z^(-${i+1}) ⋅ Y(z)`)
			.filter((t) => t != false)
		].join(' ') || 0)
	].filter((v,i) => v != 0).join(' + ')

	$: outputFormularTransfer = 'Y(z)/X(z) = (' + ([
			...a
			.map((c,i) => c==0 ? false : i > 0 ? `${signedFormatter.format(c)}⋅z^(-${i})` : `${formatter.format(c)}`)
			.filter((t) => t != false),
		].join(' ') || 0)  + ') / (1 - ' + ([
			...b
			.map((c,i) => c==0 ? false : `${(i>0?signedFormatter:formatter).format(c)}⋅z^(-${i+1})`)
			.filter((t) => t != false)
		].join(' ') || 0) + ')'

	let selectedOutput = 0
	$: outputs = [outputFormularTime,
		outputFormularFreq,
		outputFormularTransfer]

	function setOrderA(o) {
		const orderA = Math.min(Math.max(o, minOrderA), maxOrderA) + 1
		
		a = Array(orderA).fill(0).map((z,i) => a[i] || z)
		selectedPoleZero = null
	}
	
	function setOrderB(o) {
		const orderB = Math.min(Math.max(o, minOrderB), maxOrderB)
		
		b = Array(orderB).fill(0).map((z,i) => b[i] || z)
		selectedPoleZero = null
	}
	
	function inputOrderA(evt) {
		setOrderA(parseInt(evt.currentTarget.value, 10))
	}
	
	function inputOrderB(evt) {
		setOrderB(parseInt(evt.currentTarget.value, 10))
	}

	function resetInvalidA(evt) {
		const index = parseInt(evt.currentTarget.getAttribute('data-index'), 10)
		evt.currentTarget.classList.remove('error')
		evt.currentTarget.value = a[index]
	}
	
	function inputParamA(evt, scaleAll = false) {
		const v = parseFloat(evt.currentTarget.value)
		const index = parseInt(evt.currentTarget.getAttribute('data-index'), 10)

		if(isNaN(v)) {
			evt.currentTarget.classList.add('error')
			return
		} else {
			evt.currentTarget.classList.remove('error')
		}

		const lock = evt.currentTarget.parentNode.querySelector('.lock-check')
		
		scaleAll = lock && lock.checked

		if(scaleAll && a[index] != 0 && v != 0) {
			const scale = v / a[index]

			for (var i = a.length - 1; i >= 0; i--) {
				const newV = a[i] * scale

				if(i!==index) {
					a[i] = newV
				}
			}
		}

		if(a[index] !== v) {
			a[index] = v
		}
	}
	
	function inputParamB(evt, scaleAll = false) {
		const v = parseFloat(evt.currentTarget.value)
		const index = parseInt(evt.currentTarget.getAttribute('data-index'), 10)
		
		if(isNaN(v)) {
			evt.currentTarget.classList.add('error')
			return
		} else {
			evt.currentTarget.classList.remove('error')
		}
		
		if(scaleAll && b[index] !== 0) {
			const scale = v / b[index]

			b = b.map((old) => old * scale)
		} else {
			b[index] = v
		}
	}

	function isValidParamA(a, index) {
		return typeof a[index] == 'number' && !isNaN(a[index]) && (index > 0 || a[index] != 0 || !a.some((x) => x!=0))
	}

	function isValidParamB(b, index) {
		return typeof b[index] == 'number' && !isNaN(b[index])
	}
</script>

<style>
	:global(body) {
		overflow-y: scroll;
	}

	:global(.error), .invalid {
		outline: 2px solid #a00;
		background: #fee;
	}
	
	dl {
		display: grid;
		grid-template-columns: [full-start] max-content max-content max-content [full-end];
		align-items: top;
		gap: 0.2em 1em;
		margin: 0;
		justify-content: center;
	}
	
	.full {
		grid-column: full;
	}
	
	input {
		margin: 0;
		padding: 0.3em
	}
	
	dd, dt {
		margin: 0
	}
	
	dt {
		display: flex;
		align-items: start;
		padding:0.3em;
	}
	
	article {
		max-width: 60em;
		margin: auto;
	}
	
	.system {
		padding: 1em;
		display: grid;
		grid-template-columns: [in-start] 1fr [in-end] 1em [forward-start] 2fr [forward-end backward-start] 2fr [backward-end] 1em [out-start] 1fr [out-end];
		grid-template-rows: [settings-start] auto [settings-end signal-start logic-start] auto [logic-end signal-end];
		grid-auto-flow: dense;
		gap: 1em 0;
	}
	
	.feed-forward-settings {
		grid-column: forward;
		grid-row: settings;
		padding: 1em;
		padding-right: 2em;
		align-self: center;
		margin-bottom: 1em;
	}
	
	.feed-back-settings {
		grid-column: backward;
		grid-row: settings;
		padding: 1em;
		padding-right: 2em;
		align-self: center;
		margin-bottom: 1em;
	}

	.in-signal {
		grid-column: in;
		grid-row: signal;
		display: flex;
		flex-direction: column;
		align-items: stretch;
		gap: 0.5em;
		position: sticky;
		top: 1em;
		align-self: start;
	}

	.out-signal {
		grid-column: out;
		grid-row: signal;
		display: flex;
		flex-direction: column;
		align-items: stretch;
		gap: 0.5em;
		position: sticky;
		top: 1em;
		align-self: start;
	}
	
	
	.feed-forward {
		grid-area: forward;
		grid-row:  logic;
		padding-top: 3em;
	}
	
	.feed-back {
		grid-area: backward;
		grid-row:  logic;
		padding-top: 3em;
	}
	
	.stack {
		display: flex;
		flex-direction: column;
	}
	
	.stack-box {
		display: grid;
		grid-template-columns: [cover-start] 35% [center-start] 30% [center-end] 35% [cover-end];
		grid-template-rows: [cover-start center-start] 35% [center-end] 65%  [cover-end];
	}
	
	.weight-field {
		grid-area: center;
		align-self: end;
		justify-self: center;
		width: 100%;
		text-align: center;
		border: 1px solid black;
		background: white;
		padding: 0.1em 0.3em 0.2em;
	}

	.weight-input {
		width: 100%;
		margin: 0;
	}

	.signal-graph {
		display: block;
		background: #fafafa;
		margin: 0 0.1em;
		width: 10em;
		border: 1px solid #ddd;
	}

	fieldset {
		padding: 0.5em;
	}

	legend {
		background: #333;
		color: #fff;
		display: inline-block;
		padding: 0.1em 0.3em;
	}

	h3 {
		font-size: 0.9em;font-weight: bold;margin:0;
	}	

	h2 {
		font-size: 1.2em;font-weight: normal;margin:0;text-align: center;
	}

	h1 {
		text-align: center;
	}

	hr {
		justify-self: stretch; width:100%; border: none; border-top:1px solid gray;
	}

	.formular {
		font-family: monospace;height: 5em;overflow: auto;
		resize: vertical;
	}

	ul {
		list-style: none;margin:0;text-align: left;padding:0;
	}

	.system-titel {
		grid-column: forward-start / backward-end; grid-row: 2;
	}

	.pill {
		padding: 0.1em 0.5em;
		border-radius: 1em;
		margin: 0 0.2em;
		cursor: pointer;
	}

	summary {
		cursor: pointer;
	}

	details {
		max-width: 35em;
		margin: auto;
	}

	.caption {
		text-align: center;
	}

	.lock-check {
		display: none;
	}

	.lock {
		display: inline-block;
		overflow: hidden;
		margin-top: 0.1em;
		margin-bottom: 0.3em;
		width: 0.7em;
		height: 0.7em;
		background: #f0f0f0;
		cursor: pointer;
		user-select: none;
		vertical-align: top;
		color: transparent;
		padding: 0.25em;
		color: #888;
		border-radius: 10%;
	}

	.lock-check:checked + .lock {
		background: #333;
		color: #fff;
	}


	.checkbox-hidden {
		display: none;
	}

	.checkbox-row {
		display: flex;
		gap: 0.2em;
	}

	.checkbox-label {
		cursor: pointer;
		padding: 0.1em 0.5em;
		border-radius: 6px;
	}

	.checkbox-hidden:checked + .checkbox-label {
		color: #fff;
		background: #579;
	}

	.warning {
		color: #a00;
		border: 1px solid #a00;
		padding: 0.5em;
		background: #fee;
	}

	select {
		width: 100%;
		font-size: inherit;
		font: inherit;
		padding: 0;
		accent-color: red;
	}
</style>
<article>

<a href="https://tools.laszlokorte.de" title="More educational tools">More educational tools</a>

<h1>
	Signal Transformation Structure
</h1>

<p style="text-align: center;">
	Examples: 
	{#each Object.keys(examples) as x}
	<button class="pill" on:click={() => loadExample(x)}>{x}</button>
	{/each}
</p>

<details>
	<summary>Introduction</summary>

	<p>
		Below you can see a linear time invariant system. That is a system that takes a time discrete signal as an input (on the left side) and produces an resulting signal as output (on the right side). It does so in a very constrained way: the input signal can only be split, delayed, scaled and added back together. 
	</p>

	<p>
		The system is made up of two parts: Feed forward part (on the left side) create a number of copies of the input and delays each of them by one timestep. These delayed copies are then scaled individually and then summeed back together.
	</p>
	<p>
		Then in the Feed back part (on the right side) the previous result is split and delayed again but then <strong>recursively</strong> summed back together with itself. This creates a feedback loop and allowes to scale the signal exponentially.
	</p>
	<p>
		On the left side you can choose from various input signal to see the resulting output on the right side. Below the output signal graph you can see the recursive formular describing the system behaviour. This formular can be transformed into a rational function (a fraction with a polynomial in both the numerator and the denominator).
	</p>
	<p>
		The values for which the polynomials evaluate to zero are called zeros (for the numerator) and poles (for the denominator). The poles and zeroes are an alternative representation for the full system. The poles represent frequencies that amplified indefinitly when feed into the system, or in other words inputs for which the system behaviour is not stable.
	</p>
	<p>
		The poles (x) and zeros (o) are marked in the Pole/Zero plot. For the system to be stable it is required that all poles are located inside the unit circle. That is because all points outside the unit circle represent frequencies with no exponential growth. A system with poles outside the unit circle would amplify even constant or decaying input signals to produce an exponentially growing output signal. A system with poles inside the unit circle would only produce unbounded outputs for unbounded inputs (which are expected to not be given). The area beyong the outer most pole is called the regin of convergence. That is the set of a frequencies for which the system output is limited.
	</p>
</details>

<div class="system">


	<fieldset class="feed-forward-settings">
	<legend>
		Feed Forward
	</legend>
	<dl>
		<dt><label for="order">Delay count</label></dt>
		<dd><input size="4" step="1" min={minOrderA} max={maxOrderA} type="number" value={a.length-1} on:input={inputOrderA} /></dd>
		<dd class="full"><input type="range" step="1" min={minOrderA} max={maxOrderA} value={a.length-1} on:input={inputOrderA} /></dd>
	</dl>
		
	</fieldset>
	
	<fieldset class="feed-back-settings">
	<legend>
		Feed Back
	</legend>
	<dl>
		<dt><label for="order">Delay Count</label></dt>
		<dd><input size="4" step="1" min={minOrderB} max={maxOrderB} type="number" value={b.length} on:input={inputOrderB} /></dd>
		<dd class="full"><input type="range" step="1" min={minOrderB} max={maxOrderB} value={b.length} on:input={inputOrderB} /></dd>
	</dl>
	</fieldset>

	<h2 class="system-titel">System</h2>

	<div class="in-signal">
		<h2>input signal X</h2>
		<svg class="signal-graph" viewBox="-50 -50 100 100">
			<line x1="-50" x2="50" y1="0" y2="0" stroke="#aaa" vector-effect="non-scaling-stroke" />
			<line y1="-50" y2="50" x1="0" x2="0" stroke="#aaa" vector-effect="non-scaling-stroke" />
			<path d="M50,0l-4,-3v6z" fill="#aaa" />
			<path d="M0,-50l-3,4h6z" fill="#aaa" />

			<text x="5" y="-42" font-size="7" fill="#777">X[n]</text>
			<text x="47" y="-5" font-size="7" fill="#777" text-anchor="end">n</text>

			{#each input as v,i}
			<line y1="0" y2="{-v*25}" x1="{(i-input.length/2) * 90/input.length}" x2="{(i-input.length/2) * 90/input.length}" stroke="#0af" vector-effect="non-scaling-stroke" stroke-width="2" />
			<circle fill="#0af" r="1" cy="{-v*25}" cx="{(i-input.length/2) * 90/input.length}" />
			{/each}
		</svg>
		<select bind:value={inputName}>
			{#each Object.keys(inputs) as name}
			<option>{name}</option>
			{/each}
		</select>
	</div>

	<div class="out-signal">
		<h2>output signal Y</h2>
		<svg class="signal-graph" viewBox="-50 -50 100 100">
			<line x1="-50" x2="50" y1="0" y2="0" stroke="#aaa" vector-effect="non-scaling-stroke" />
			<line y1="-50" y2="50" x1="0" x2="0" stroke="#aaa" vector-effect="non-scaling-stroke" />
			<path d="M50,0l-4,-3v6z" fill="#aaa" />
			<path d="M0,-50l-3,4h6z" fill="#aaa" />
			<text x="5" y="-42" font-size="7" fill="#777">Y[n]</text>
			<text x="47" y="-5" font-size="7" fill="#777" text-anchor="end">n</text>

			{#each output as v,i}
			<line y1="0" y2="{-v*25}" x1="{(i-output.length/2) * 90/output.length}" x2="{(i-output.length/2) * 90/output.length}" stroke="#0a6" vector-effect="non-scaling-stroke" stroke-width="2" />
			<circle fill="#0a6" r="1" cy="{-v*25}" cx="{(i-output.length/2) * 90/output.length}" />
			{/each}
		</svg>

		<h2>Output Function</h2>

		<div class="checkbox-row">
			<label><input class="checkbox-hidden" type="radio" bind:group={selectedOutput} value={0} /> <span class="checkbox-label">Time</span></label>
			<label><input class="checkbox-hidden" type="radio" bind:group={selectedOutput} value={1} /> <span class="checkbox-label">Freq</span></label>
			<label><input class="checkbox-hidden" type="radio" bind:group={selectedOutput} value={2} /> <span class="checkbox-label">Transfere</span></label>
		</div>

		<textarea readonly class="formular">{outputs[selectedOutput]}</textarea>

		<hr>
		<h2>Pole/Zero Plot</h2>

		{#if unsolvable}
			<div class="warning">
				The Amplificiation a<sub>0</sub> must not be zero. Otherwise the function can not be factored into a pole/zero representation.
			</div>
		{:else}
			<svg class="signal-graph" viewBox="-50 -50 100 100" on:click={()=>{selectedPoleZero = null}}>
				<line x1="-50" x2="50" y1="0" y2="0" stroke="#aaa" vector-effect="non-scaling-stroke" />
				<line y1="-50" y2="50" x1="0" x2="0" stroke="#aaa" vector-effect="non-scaling-stroke" />
				<path d="M50,0l-4,-3v6z" fill="#aaa" />
				<path d="M0,-50l-3,4h6z" fill="#aaa" />

				<text x="5" y="-42" font-size="7" fill="#777">Im</text>
				<text x="47" y="-5" font-size="7" fill="#777" text-anchor="end">Re</text>

				<circle cx="0" cy="0" r="20" stroke="#aaa" fill="none" vector-effect="non-scaling-stroke" />

				<path fill-rule="evenodd" d="M-50,-50L50,-50L50,50L-50,50z M{-rocRadiusMin * 20},0a{rocRadiusMin * 20} {rocRadiusMin * 20} 0 1 1 0 1z" fill="lime" fill-opacity="0.1" />
				<circle cx="0" cy="0" r="{rocRadiusMin * 20}" stroke-width="1" stroke="green" fill="none" stroke-dasharray="3 3" vector-effect="non-scaling-stroke" />

				{#if zeros.length}
				{#each zeros as [zx,zy], i}
					<circle cx={zx*20} cy={zy*20} r="2" fill="none" stroke="red" vector-effect="non-scaling-stroke" stroke-width={selectedPoleZero == `z-${i}` ? 3 : 1}>
						<title>Root</title>
					</circle>
				{/each}
				{/if}
				{#if poles.length}
				{#each poles as [px,py], i}
				 <path d="M{svgNumber.format(px*20)}, {svgNumber.format(py*20)}m-2,-2l4,4m-4,0l4,-4" vector-effect="non-scaling-stroke" stroke="blue" stroke-width={selectedPoleZero == `p-${i}` ? 3 : 1}>
				 	<title>Pole</title>
				 </path>
				{/each}
				{/if}
			</svg>

			<div class="caption">
				{#if stable}
				Stable
				{:else}
				Unstable
				{/if}
			</div>

			<div>
				<h3>Gain</h3>
				<ul>
					<li>{a[0]}</li>
				</ul>
				<h3>Poles</h3>
				<select size="3" bind:value={selectedPoleZero}>
					{#each poles as p, i}
					 <option value="p-{i}">{formatComplex(p)}</option>
					{/each}
				</select>

				<h3>Zeros</h3>
				<select size="3" bind:value={selectedPoleZero}>
					{#each zeros as z, i}
					 <option value="z-{i}">{formatComplex(z)}</option>
					{/each}
				</select>
			</div>
		{/if}

		
	</div>


	<div class="feed-forward stack">
		{#each a as v,i}
		<div class="stack-box">
			<Mesh first={i==0} last={i>=a.length-1} />
			<div class="weight-field">
				<label for="a_{i}">Amplify</label>
				{#if i==0}
				<input id="lock" class="lock-check" type="checkbox" />
				<label title="Lock Ratio" for="lock" class="lock">
					<Link />
				</label>
				{/if}
				<input class:invalid={!isValidParamA(a,i)} min="-10" max="10" step="0.01" class="weight-input" id="a_{i}" data-index={i} type="number" value={formatter.format(v)} size="3" on:input={inputParamA} on:blur={resetInvalidA} />
			</div>
		</div>
		{/each}
	</div>
	<div class="feed-back stack">
		<div class="stack-box">
			<Mesh first={true} last="{b.length<1}" back out />
		</div>
		{#each b as v,i}
		<div class="stack-box">
			<Mesh first={false} last={i>=b.length-1} out back />
			
			<div class="weight-field">
			<label for="b_{i}">Amplify</label>
			<input class:invalid={!isValidParamB(b,i)} min="-10" max="10" step="0.01" class="weight-input" id="b_{i}" data-index={i} type="number" value={formatter.format(v)} size="3" on:input={inputParamB} />
			</div>		
		</div>
		{/each}
	</div>
</div>
</article>