<script>
	import Mesh from './Mesh.svelte'
	import durand from 'durand-kerner'
	
	const formatter = new Intl.NumberFormat('en-US', { maximumFractionDigits: 2, minimumFractionDigits: 2 });
	const signedFormatter = new Intl.NumberFormat('en-US', { maximumFractionDigits: 2, minimumFractionDigits: 2, 'signDisplay': 'always' });
	const svgNumber = new Intl.NumberFormat('en-US', { maximumFractionDigits: 2, minimumFractionDigits: 2 });
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
		'Unstable': {
			a: [.2,-.3,0.4,-0.1],
			b: [-1.1,0.2,0.7],
		},
	}

	function loadExample(name) {
		a = examples[name].a
		b = examples[name].b
	}

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

		return durand(arr)
	}

	$: roots = findRoots(a.map(v=>v).reverse())
	$: poles = findRoots([-1,...b].map((v)=>-v).reverse())

	$: poleMagnitudesSquares = poles[0] ? poles[0].map((_,i) => poles[0][i]*poles[0][i] + poles[1][i]*poles[1][i]) : []

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
	$: outputFormular = 'y[n] = ' + [
		...a
		.map((c,i) => c==0 ? false : i > 0 ? `${signedFormatter.format(c)}*x[n-${i}]` : `${formatter.format(c)}*x[n]`)
		.filter((t) => t != false),
		...b
		.map((c,i) => c==0 ? false : i > 0 ? `${signedFormatter.format(c)}*y[n-${i+1}]` : `${formatter.format(c)}*y[n-1]`)
		.filter((t) => t != false)
	].join(' ')

	function setOrderA(o) {
		const orderA = Math.min(Math.max(o, minOrderA), maxOrderA) + 1
		
		a = Array(orderA).fill(0).map((z,i) => a[i] || z)
	}
	
	function setOrderB(o) {
		const orderB = Math.min(Math.max(o, minOrderB), maxOrderB)
		
		b = Array(orderB).fill(0).map((z,i) => b[i] || z)
	}
	
	function inputOrderA(evt) {
		setOrderA(parseInt(evt.currentTarget.value, 10))
	}
	
	function inputOrderB(evt) {
		setOrderB(parseInt(evt.currentTarget.value, 10))
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
		
		if(scaleAll && a[index] !== 0) {
			const scale = v / a[index]

			a = a.map((old) => old * scale)
		} else {
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
</script>

<style>
	:global(.error) {
		outline: 2px solid red;
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
		font-family: monospace;height: 7em;overflow: auto;
	}

	ul {
		list-style: none;margin:0;text-align: left;padding:0;
	}

	.system-titel {
		grid-column: forward-start / backward-end; grid-row: 2;
	}
</style>
<article>

<a href="https://tools.laszlokorte.de" title="More educational tools">More educational tools</a>

<h1>
	Signal Transformation Structure
</h1>

<p style="text-align: center;">Examples: {#each Object.keys(examples) as x}<button on:click={() => loadExample(x)}>{x}</button>{/each}</p>

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

			{#each output as v,i}
			<line y1="0" y2="{-v*25}" x1="{(i-output.length/2) * 90/output.length}" x2="{(i-output.length/2) * 90/output.length}" stroke="#0a6" vector-effect="non-scaling-stroke" stroke-width="2" />
			<circle fill="#0a6" r="1" cy="{-v*25}" cx="{(i-output.length/2) * 90/output.length}" />
			{/each}
		</svg>

		<div class="formular">
			{outputFormular}
		</div>
		<hr>
		<h2>Pole/Zero Plot</h2>
		<svg class="signal-graph" viewBox="-50 -50 100 100">
			<line x1="-50" x2="50" y1="0" y2="0" stroke="#aaa" vector-effect="non-scaling-stroke" />
			<line y1="-50" y2="50" x1="0" x2="0" stroke="#aaa" vector-effect="non-scaling-stroke" />
			<path d="M50,0l-4,-3v6z" fill="#aaa" />
			<path d="M0,-50l-3,4h6z" fill="#aaa" />

			<circle cx="0" cy="0" r="20" stroke="#ddd" fill="none" stroke-dasharray="5 5" />
			<circle cx="0" cy="0" r="{rocRadiusMin * 20}" stroke="cyan" fill="none" stroke-dasharray="1 1" />

			{#if roots.length}
			{#each roots[0] as r, i}
			<circle cx={r*20} cy={roots[1][i]*20} r="2" fill="none" stroke="blue" vector-effect="non-scaling-stroke"  />
			{/each}
			{/if}
			{#if poles.length}
			{#each poles[0] as r, i}
			 <path d="M{svgNumber.format(r*20)}, {svgNumber.format(poles[1][i]*20)}m-2,-2l4,4m-4,0l4,-4" vector-effect="non-scaling-stroke" stroke="red" />
			{/each}
			{/if}
		</svg>

		{#if stable}
		Stable
		{:else}
		Unstable
		{/if}

		<div>
			<h3>Poles</h3>
			{#if poles[0]}
			<ul>
				{#each poles[0] as r, i}
				 <li>{formatter.format(poles[0][i])}{signedFormatter.format(poles[1][i])}i</li>
				{/each}
			</ul>
			{:else}
			None
			{/if}

			<h3>Roots</h3>
			{#if roots[0]}
			<ul >
				{#each roots[0] as r, i}
				 <li>{formatter.format(roots[0][i])}{signedFormatter.format(roots[1][i])}i</li>
				{/each}
			</ul>
			{:else}
			None
			{/if}
		</div>
	</div>


	<div class="feed-forward stack">
		{#each a as v,i}
		<div class="stack-box">
			<Mesh first={i==0} last={i>=a.length-1} />
			<div class="weight-field">
				<label for="a_{i}">Gain</label>
			<input min="-10" max="10" step="0.1" class="weight-input" id="a_{i}" data-index={i} type="number" value={(v)} size="3" on:input={inputParamA} />
			
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
			<label for="b_{i}">Gain</label>
			<input min="-10" max="10" step="0.01" class="weight-input" id="b_{i}" data-index={i} type="number" value={(v)} size="3" on:input={inputParamB} />
			</div>		
		</div>
		{/each}
	</div>
</div>
</article>