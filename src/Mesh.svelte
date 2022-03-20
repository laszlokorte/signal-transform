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
	}
	
	.hidden {
		display: none;
	}
</style>

<svg viewBox="-30 0 360 100" class="stack-mesh">
	<path vector-effect="non-scaling-stroke" class:hidden={!debug} id="placeolder" d="M159,16C159,14 157,12 154,12L145,12C142,12 140,14 140,16L140,26C140,28 142,31 145,31L154,31C157,31 159,28 159,26L159,16Z" fill="red" />
	<path vector-effect="non-scaling-stroke" class:hidden={!first} id="long" d="M-50,40L330,40" fill="none" stroke="black" stroke-width="1" />
	<path vector-effect="non-scaling-stroke" class:hidden={first} id="up" d="M259,-10L259,40L32,40L32,-10" fill="none" stroke="black" stroke-width="1" />
	<path vector-effect="non-scaling-stroke" class:hidden={last} id="down" d="M259,100L259,40M32,40L32,100" fill="none" stroke="black" stroke-width="1" />
	
	<path vector-effect="non-scaling-stroke" class:hidden={!out || !first} id="right-out" d="M330,40L322,43L322,37L330,40Z"/>
	
	
	<g id="back" class:hidden={!back}>
		<circle class:hidden={last}  id="plus-circle-back" cx="32" cy="40" r="12" fill="white" stroke="black" stroke-width="1" />

		<text class:hidden={last} x="32" y="42" text-anchor="middle" dominant-baseline="middle">+</text>

		<path vector-effect="non-scaling-stroke" class:hidden={last || first} id="right-arrow-back" d="M44,40L52,43L52,37L44,40Z"/>    
	
		<path vector-effect="non-scaling-stroke" class:hidden={last || !first} id="left-arrow" d="M20,40L12,37L12,43L20,40Z"/>

		<path vector-effect="non-scaling-stroke" class:hidden={last} id="up-arrow-back" d="M32,52L29,59L35,59L32,52Z"/>
	</g>
		
	<g class:hidden={last || !back} id="delay-right">
		<rect x="243" y="68" width="31" height="31" fill="white" stroke="black" stroke-width="1" />
		<text x="261" y="86" text-anchor="middle" dominant-baseline="middle">z<tspan dy ="-7" font-size="12">-1</tspan></text>
	</g>
	
	<g class:hidden={!first || out} id="in">
		<rect x="-29" y="23" width="40" height="31" fill="white" stroke="black" stroke-width="1" />
		<text x="-8" y="41" text-anchor="middle" dominant-baseline="middle">X[n]</text>
	</g>
	
	<g class:hidden={last || back} id="delay">
		<rect x="15" y="66" width="31" height="31" fill="white" stroke="black" stroke-width="1" />
		<text x="32" y="84" text-anchor="middle" dominant-baseline="middle">z<tspan dy ="-7" font-size="12">-1</tspan></text>
	</g>
	<g class:hidden={back}>
		<circle  class:hidden={last} id="plus-circle" cx="259" cy="40" r="12" fill="white" stroke="black" stroke-width="1" />
		<text class:hidden={last} x="260" y="42" text-anchor="middle" dominant-baseline="middle">+</text>
		<path vector-effect="non-scaling-stroke" class:hidden={last} id="right-arrow" d="M247,40L239,43L239,37L247,40Z"/>
		<path vector-effect="non-scaling-stroke" class:hidden={last} id="up-arrow" d="M259,52L262,59L256,59L259,52Z"/>
	</g>
</svg>