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
	}
	
	.hidden {
		display: none;
	}
</style>

<svg viewBox="-30 0 360 100" class="stack-mesh">
	<path vector-effect="non-scaling-stroke" class:hidden={!debug} id="placeolder" d="M159.5,16.75C159.5,14.128 157.372,12 154.75,12L145.25,12C142.628,12 140.5,14.128 140.5,16.75L140.5,26.25C140.5,28.872 142.628,31 145.25,31L154.75,31C157.372,31 159.5,28.872 159.5,26.25L159.5,16.75Z" style="fill:rgb(255,0,0);"/>
	<path vector-effect="non-scaling-stroke" class:hidden={!first} id="long" d="M-50,40L330,40" style="fill:none;stroke:black;stroke-width:1px;"/>
	<path vector-effect="non-scaling-stroke" class:hidden={first} id="up" d="M259,-10L259,40L32,40L32,-10" style="fill:none;stroke:black;stroke-width:1px;"/>
	<path vector-effect="non-scaling-stroke" class:hidden={last} id="down" d="M259,100L259,40M32,40L32,100" style="fill:none;stroke:black;stroke-width:1px;"/>
	
	<path vector-effect="non-scaling-stroke" class:hidden={!out || !first} id="right-out" d="M330,40L322,43L322,37L330,40Z"/>
	
	
	<g id="back" class:hidden={!back}>
		<circle class:hidden={last}  id="plus-circle-back" cx="32" cy="40" r="12" style="fill:white;stroke:black;stroke-width:1px;"/>
		<path vector-effect="non-scaling-stroke"  class:hidden={last} id="plus-sign-back" d="M32.673,43.839L32.673,40.654L35.835,40.654L35.835,39.323L32.673,39.323L32.673,36.161L31.327,36.161L31.327,39.323L28.165,39.323L28.165,40.654L31.327,40.654L31.327,43.839L32.673,43.839Z" style="fill-rule:nonzero;"/>
		<path vector-effect="non-scaling-stroke" class:hidden={last || first} id="right-arrow-back" d="M44,40L52,43L52,37L44,40Z"/>    
	
		<path vector-effect="non-scaling-stroke" class:hidden={last || !first} id="left-arrow" d="M20,40L12,37L12,43L20,40Z"/>

		<path vector-effect="non-scaling-stroke" class:hidden={last} id="up-arrow-back" d="M32,52L29,59L35,59L32,52Z"/>
	</g>
		
	<g class:hidden={last || !back} id="delay-right">
		<rect x="243" y="68" width="31" height="31" style="fill:white;stroke:black;stroke-width:1px;"/>
		<text x="261" y="86" text-anchor="middle" dominant-baseline="middle">z<tspan dy ="-7" font-size="12">-1</tspan></text>
	</g>
	
	<g class:hidden={!first || out} id="in">
		<rect x="-29" y="23" width="40" height="31" style="fill:white;stroke:black;stroke-width:1px;"/>
		<text x="-8" y="41" text-anchor="middle" dominant-baseline="middle">X[n]</text>
	</g>
	
	<g class:hidden={last || back} id="delay">
		<rect x="15" y="66" width="31" height="31" style="fill:white;stroke:black;stroke-width:1px;"/>
		<text x="32" y="84" text-anchor="middle" dominant-baseline="middle">z<tspan dy ="-7" font-size="12">-1</tspan></text>
	</g>
	<g class:hidden={back}>
		<circle  class:hidden={last} id="plus-circle" cx="259" cy="40" r="12" style="fill:white;stroke:black;stroke-width:1px;"/>
		<path vector-effect="non-scaling-stroke"  class:hidden={last} id="plus-sign" d="M258.327,43.839L258.327,40.654L255.165,40.654L255.165,39.323L258.327,39.323L258.327,36.161L259.673,36.161L259.673,39.323L262.835,39.323L262.835,40.654L259.673,40.654L259.673,43.839L258.327,43.839Z" style="fill-rule:nonzero;"/>
		<path vector-effect="non-scaling-stroke" class:hidden={last} id="right-arrow" d="M247,40L239,43L239,37L247,40Z"/>
		<path vector-effect="non-scaling-stroke" class:hidden={last} id="up-arrow" d="M259,52L262,59L256,59L259,52Z"/>
	</g>
</svg>