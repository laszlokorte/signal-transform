<script> 
export let back = false;
export let out = false;
 export let first = false;
 export let last = false;
 export let debug = false;
</script>

<style>
	.stack-mesh {
		grid-area: cover;
		width: 100%;
		display: block;
		margin: 0;
		align-self: stretch;
	}
	
	.hidden {
		display: none;
	}
</style>

<svg viewBox="-30 0 360 100" class="stack-mesh" preserveAspectRatio="xMidYMin meet">
	<path vector-effect="non-scaling-stroke" class:hidden={!debug} d="M159,16C159,14 157,12 154,12L145,12C142,12 140,14 140,16L140,26C140,28 142,31 145,31L154,31C157,31 159,28 159,26L159,16Z" fill="red" />
	<path vector-effect="non-scaling-stroke" class:hidden={!first} d="M-50,40L330,40" fill="none" stroke="black" stroke-width="1" shape-rendering="optimizeSpeed"
 />
	<path vector-effect="non-scaling-stroke" class:hidden={first} d="M259,-10L259,40L32,40L32,-10" fill="none" stroke="black" stroke-width="1" shape-rendering="optimizeSpeed"
 />
	<path vector-effect="non-scaling-stroke" class:hidden={last} d="M259,200L259,40M32,40L32,200" fill="none" stroke="black" stroke-width="1" shape-rendering="optimizeSpeed"
 />
	
	<path vector-effect="non-scaling-stroke" class:hidden={!out || !first} d="M330,40L322,45L322,35L330,40Z"/>
	
	
	<g class:hidden={!back}>
		<g>
			<title>Sum Signals</title>
			<circle class:hidden={last} vector-effect="non-scaling-stroke" cx="32" cy="40" r="12" fill="white" stroke="black" stroke-width="1"
 />
			<text class:hidden={last} x="32" y="41.5" text-anchor="middle" dominant-baseline="middle" font-size="24">+</text>
		</g>

		<path vector-effect="non-scaling-stroke" class:hidden={last || first} d="M44,40L52,45L52,35L44,40Z"/>    
	
		<path vector-effect="non-scaling-stroke" class:hidden={last || !first} d="M20,40L12,35L12,45L20,40Z"/>

		<path vector-effect="non-scaling-stroke" class:hidden={last} d="M32,52L27,59L37,59L32,52Z"/>
	</g>
		
	<g class:hidden={last || !back}>
		<title>Delay by one sample</title>
		<rect vector-effect="non-scaling-stroke" x="243" y="68" width="31" height="31" fill="white" stroke="black" stroke-width="1" 
		shape-rendering="optimizeSpeed"
 />
		<text x="261" y="86" text-anchor="middle" dominant-baseline="middle">z<tspan dy ="-7" font-size="12">-1</tspan></text>
	</g>
	
	<g class:hidden={!first || out}>
		<title>Input signal</title>
		<rect vector-effect="non-scaling-stroke" x="-29" y="23" width="40" height="31" fill="white" stroke="black" stroke-width="1" shape-rendering="optimizeSpeed"
 />
		<text x="-8" y="41" text-anchor="middle" dominant-baseline="middle">x[n]</text>
	</g>
	
	<g class:hidden={last || back}>
		<title>Delay by one sample</title>
		<rect vector-effect="non-scaling-stroke" x="15" y="66" width="31" height="31" fill="white" stroke="black" stroke-width="1" shape-rendering="optimizeSpeed"
 />
		<text x="32" y="84" text-anchor="middle" dominant-baseline="middle">z<tspan dy ="-7" font-size="12">-1</tspan></text>
	</g>
	<g class:hidden={back}>
		<g>
			<title>Sum Signals</title>
			<circle vector-effect="non-scaling-stroke" class:hidden={last} cx="259" cy="40" r="12" fill="white" stroke="black" stroke-width="1"
 />
			<text class:hidden={last} x="259" y="41.5" text-anchor="middle" dominant-baseline="middle" font-size="24">+</text>
		</g>
		<path vector-effect="non-scaling-stroke" class:hidden={last} d="M247,40L239,45L239,35L247,40Z"/>
		<path vector-effect="non-scaling-stroke" class:hidden={last} d="M259,52L264,59L254,59L259,52Z"/>
	</g>
</svg>