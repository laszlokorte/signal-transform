<script>
	import Mesh from "./Mesh.svelte";
	import Link from "./Link.svelte";
	import durand from "durand-kerner";

	const formatter = new Intl.NumberFormat("en-US", {
		maximumFractionDigits: 2,
		minimumFractionDigits: 2,
	});
	const signedFormatter = new Intl.NumberFormat("en-US", {
		maximumFractionDigits: 2,
		minimumFractionDigits: 2,
		signDisplay: "always",
	});
	const svgNumber = new Intl.NumberFormat("en-US", {
		maximumFractionDigits: 2,
		minimumFractionDigits: 2,
	});

	function formatComplex(c, polar) {
		if (polar) {
			const phase = signedFormatter.format(Math.atan2(c[1], c[0]));
			const magnitude = formatter.format(
				Math.sqrt(c[0] * c[0] + c[1] * c[1]),
			);
			if (magnitude == "0.00") {
				return "0.00";
			}
			return `${magnitude}⋅e^(${phase}i)`;
		} else {
			const re = formatter.format(c[0]);
			const im = signedFormatter.format(c[1]);
			return `${re == "-0.00" ? "0.00" : re}${im == "-0.00" ? "+0.00" : im}i`;
		}
	}

	const sampleRate = 40;

	const examples = {
		Identity: {
			b: [1],
			a: [],
			causal: true,
		},
		"Low pass": {
			b: [0.25, 0.25],
			a: [1, -0.5],
			causal: true,
		},
		"High pass": {
			b: [1, -1],
			a: [],
			causal: true,
		},
		Unstable: {
			b: [0.2, -0.3, 0.4, -0.1],
			a: [-1.4, 0.01, 0.9],
			causal: true,
		},
	};

	function loadExample(name) {
		b = examples[name].b.slice();
		a = examples[name].a.slice();
		const [p, z] = toPoleZero(b, a, examples[name].causal);
		poles = p;
		zeros = z;
		causal = !!examples[name].causal;
		selectedPoleZero = null;
	}

	let selectedPoleZero = null;

	let causal = true;

	let b = [1];
	const minOrderB = 0;
	const maxOrderB = Math.floor(sampleRate / 2) - 1;

	let a = [];
	const minOrderA = 0;
	const maxOrderA = Math.floor(sampleRate / 2);

	let gain = 1;
	let zeros = [];
	let poles = [];

	let pzSvg = null;
	$: svgPoint = pzSvg && pzSvg.createSVGPoint();

	function findRoots(arrO) {
		const arr = arrO.map((x) => x);
		while (arr[0] == 0) {
			arr.shift();
		}

		let leadings = 0;
		while (arr[arr.length - 1] == 0) {
			leadings++;
			arr.pop();
		}

		const roots = durand(arr);

		const result =
			roots.length && roots[0].length
				? roots[0].map((_, i) => [roots[0][i], roots[1][i]])
				: [];

		return [...result, ...Array(leadings).fill([0, 0])];
	}

	function compareComplex(a, b) {
		return (
			Math.sign(
				Math.round(
					10 * (b[0] * b[0] + b[1] * b[1]) -
						10 * (a[0] * a[0] + a[1] * a[1]),
				),
			) || Math.sign(Math.atan2(b[0], b[1]) - Math.atan2(a[0], a[1]))
		);
	}

	function toPoleZero(b, a, causal) {
		const zeros = findRoots(causal ? b.map((v) => v).reverse() : b);
		const poles = findRoots(
			causal
				? [-1, ...a].map((v) => -v).reverse()
				: [-1, ...a].map((v) => -v),
		);

		const polynomeOrder = Math.max(zeros.length, poles.length);

		while (zeros.length < polynomeOrder) {
			zeros.push(causal ? [0, 0] : [Infinity, Infinity]);
		}

		while (poles.length < polynomeOrder) {
			poles.push(causal ? [0, 0] : [Infinity, Infinity]);
		}

		return [poles, zeros];
	}

	$: unsolvable = false && b[0] == 0 && b.some((v) => v != 0);

	$: poleMagnitudesSquares = poles.map((p) => p[0] * p[0] + p[1] * p[1]);

	$: rocRadiusMin = Math.sqrt(Math.max(0, ...poleMagnitudesSquares));
	$: rocRadiusMax = Math.sqrt(Math.min(9999, ...poleMagnitudesSquares));
	$: stable = causal ? rocRadiusMin < 1 : rocRadiusMax > 1;

	function selectionToPos(p, z, sel) {
		if (p.length && sel && sel.substr(0, 1) == "p") {
			const idx = parseInt(sel.substr(2), 10);
			if (idx < p.length) {
				return p[idx];
			}
		} else if (z.length && sel && sel.substr(0, 1) == "z") {
			const idx = parseInt(sel.substr(2), 10);
			if (idx < z.length) {
				return z[idx];
			}
		}

		return null;
	}

	let dragState = null;

	function dragStart(evt) {
		evt.stopPropagation();
		const selected = evt.currentTarget.getAttribute("data-polezero");
		const type = selected.substr(0, 1);
		const idx = parseInt(selected.substr(2));
		const all = type == "z" ? zeros : poles;

		const self = all[idx];
		const sign = Math.sign(Math.round(self[1] * 1000));
		const others = all.filter((_, i) => i !== idx);
		const keep = others.filter(
			(o) =>
				Math.sign(Math.round(o[1] * 1000)) == sign ||
				Math.sign(Math.round(o[1] * 1000)) == 0 ||
				(sign == 0 && Math.sign(Math.round(o[1] * 1000)) > 0),
		);

		dragState = {
			type: type,
			idx: idx,
			keep: keep,
			sign: sign,
		};
	}

	function dragUpdate(evt) {
		if (!dragState || !pzSvg || !svgPoint) {
			return;
		}

		svgPoint.x = evt.clientX;
		svgPoint.y = evt.clientY;
		const pos = svgPoint.matrixTransform(pzSvg.getScreenCTM().inverse());

		const selected = selectedPoleZero;
		const type = dragState.type;
		const idx = dragState.idx;
		const keep = dragState.keep;
		const sign = dragState.sign;
		const all = type == "z" ? zeros : poles;

		if (idx > all.length - 1) {
			return;
		}

		const self = all[idx];
		const newValue = [
			Math.max(-2.5, Math.min(2.5, pos.x / 20)),
			Math.max(-2.5, Math.min(2.5, (Math.abs(sign) * pos.y) / -20)),
		];

		const newValues = [...keep, newValue].flatMap(([re, im]) =>
			Math.sign(Math.round(im * 1000)) == 0
				? [[re, 0]]
				: [
						[re, -im],
						[re, im],
					],
		);

		const coefs = rootsToCoefs(newValues)
			.map((z) => z[0])
			.map(
				(x) =>
					Math.pow(
						-1,
						Math.max(all.length, newValues.length) -
							Math.min(all.length, newValues.length) +
							(type == "z" ? 0 : 1),
					) * x,
			);

		const newIdx = newValues.length - 1;

		if (type == "z") {
			if (newValues.length == zeros.length) {
				b = causal
					? coefs.map((k) => k * Math.abs(b[0]) * Math.sign(coefs[0]))
					: coefs.reverse();
				selectedPoleZero = `z-${newIdx}`;
				const extraPoles = Array(
					Math.max(0, newValues.length - poles.length),
				).fill(causal ? [0, 0] : [Infinity, Infinity]);

				zeros = newValues;
				poles = [...poles, ...extraPoles];
			}
		} else {
			if (newValues.length == poles.length) {
				a = causal
					? coefs.slice(1).map((k) => -k / coefs[0])
					: coefs
							.slice(1)
							.map((k) => coefs[0] / -k)
							.reverse();

				selectedPoleZero = `p-${newIdx}`;
				const extraZeros = Array(
					Math.max(0, newValues.length - zeros.length),
				).fill(causal ? [0, 0] : [Infinity, Infinity]);

				poles = newValues;
				zeros = [...zeros, ...extraZeros];
			}
		}
	}

	function dragStop(evt) {
		if (!dragState) {
			return;
		}
		evt.preventDefault();
		evt.stopPropagation();
		dragState = null;
	}

	function complexMul(a, b) {
		return [a[0] * b[0] - a[1] * b[1], a[0] * b[1] + a[1] * b[0]];
	}

	function complexAdd(a, b) {
		return [a[0] + b[0], a[1] + b[1]];
	}

	function rootsToCoefs(roots) {
		const o = [...roots, 0].fill([0, 0]);
		for (let i = 0; i < 2 ** roots.length; i++) {
			let a = [1, 0];
			let t_index = roots.length;
			for (let j = 0; j < roots.length; j++) {
				if (i & (1 << j)) {
					a = complexMul(a, roots[j]);
				} else {
					a = complexMul(a, [-1, 0]);
					t_index--;
				}
			}
			o[t_index] = complexAdd(o[t_index], a);
		}
		return o;
	}

	function updatePole(evt) {
		const selected = evt.currentTarget.getAttribute("data-polezero");
		const type = selected.substr(0, 1);
		const idx = parseInt(selected.substr(2));
		const all = type == "z" ? zeros : poles;

		const self = all[idx];
		const sign = Math.sign(Math.round(self[1] * 1000));
		const others = all.filter((_, i) => i !== idx);
		const keep = others.filter(
			(o) =>
				Math.sign(Math.round(o[1] * 1000)) == sign ||
				Math.sign(Math.round(o[1] * 1000)) == 0 ||
				(sign == 0 && Math.sign(Math.round(o[1] * 1000)) > 0),
		);

		const i = parseInt(evt.currentTarget.getAttribute("data-i"), 10);

		const newValue = self.slice();
		newValue[i] =
			Math.round(parseFloat(evt.currentTarget.value) * 1000) / 1000;

		const newValues = [...keep, newValue].flatMap(([re, im]) =>
			Math.sign(Math.round(im * 1000)) == 0
				? [[re, 0]]
				: [
						[re, -im],
						[re, im],
					],
		);

		const coefs = rootsToCoefs(newValues)
			.map((z) => z[0])
			.map(
				(x) =>
					Math.pow(
						-1,
						Math.max(all.length, newValues.length) -
							Math.min(all.length, newValues.length) +
							(type == "z" ? 0 : 1),
					) * x,
			);

		const newIdx = newValues.length - 1;

		if (type == "z") {
			b = coefs.map((k) => k * Math.abs(b[0]) * Math.sign(coefs[0]));
			selectedPoleZero = `z-${newIdx}`;
			const extraPoles = Array(
				Math.max(0, newValues.length - poles.length),
			).fill([0, 0]);
			zeros = newValues;
			poles = [...poles, ...extraPoles];
		} else {
			a = coefs.slice(1).map((k) => -k / coefs[0]);
			selectedPoleZero = `p-${newIdx}`;
			const extraZeros = Array(
				Math.max(0, newValues.length - zeros.length),
			).fill([0, 0]);
			poles = newValues;
			zeros = [...zeros, ...extraZeros];
		}
	}

	$: selectedPoleZeroPos = selectionToPos(poles, zeros, selectedPoleZero);

	let inputName = "impulse";
	const inputs = {
		impulse: Array(sampleRate)
			.fill(0)
			.map((v, i) => (i == Math.round(sampleRate / 2) ? 1 : v)),
		"impulse train": Array(sampleRate)
			.fill(0)
			.map((v, i) => (i % Math.round(sampleRate / 8) == 0 ? 1 : v)),
		sin: Array(sampleRate)
			.fill(0)
			.map((v, i) => Math.sin(2 * Math.PI * (i / sampleRate + 0.5))),
		cos: Array(sampleRate)
			.fill(0)
			.map((v, i) => Math.cos(2 * Math.PI * (i / sampleRate + 0.5))),
		const: Array(sampleRate)
			.fill(0)
			.map((v, i) => 1),
		step: Array(sampleRate)
			.fill(0)
			.map((v, i) => (i < sampleRate / 2 ? 0 : 1)),
		relu: Array(sampleRate)
			.fill(0)
			.map((v, i) =>
				i < sampleRate / 2 ? 0 : 2 * ((2 * i) / sampleRate - 1),
			),
		triangle: Array(sampleRate)
			.fill(0)
			.map((v, i) =>
				Math.abs(i - sampleRate / 2) > sampleRate / 4
					? 0
					: 0.1 *
						Math.abs(sampleRate / 4 - Math.abs(i - sampleRate / 2)),
			),
	};

	$: input = inputs[inputName];

	function getOutputAt(input, b, a, i, cache = [], causality = false) {
		if (causality) {
			if (i < 0) return 0;
			if (cache[i] === undefined)
				cache[i] =
					b.reduce(
						(sum, bb, j) => sum + bb * (input[i - j] || 0),
						0,
					) +
					a.reduce(
						(sum, aa, k) =>
							sum +
							aa *
								getOutputAt(
									input,
									b,
									a,
									i - k - 1,
									cache,
									causality,
								),
						0,
					);

			return cache[i];
		} else {
			if (i >= input.length) return 0;
			if (cache[i] === undefined)
				cache[i] =
					b.reduce(
						(sum, bb, j) => sum + bb * (input[i + j + 1] || 0),
						0,
					) +
					a.reduce(
						(sum, aa, k) =>
							sum +
							aa *
								getOutputAt(
									input,
									b,
									a,
									i + k + 1,
									cache,
									causality,
								),
						0,
					);
			return cache[i];
		}
	}

	function getImpulseResponseAt(b, a, i, cache = [], causality = false) {
		if (causality) {
			if (i < 0) return 0;
			if (cache[i] === undefined) {
				cache[i] =
					b.reduce((sum, bb, j) => sum + bb * (i == j), 0) +
					a.reduce(
						(sum, aa, k) =>
							sum +
							aa *
								getOutputAt(
									input,
									b,
									a,
									i - k - 1,
									cache,
									causality,
								),
						0,
					);
			}

			return cache[i];
		} else {
			if (i >= 0) return 0;
			if (cache[i] === undefined) {
				cache[i] =
					b.reduce((sum, bb, j) => sum + bb * (i == j), 0) +
					a.reduce(
						(sum, aa, k) =>
							sum +
							aa *
								getOutputAt(
									input,
									b,
									a,
									i - k - 1,
									cache,
									causality,
								),
						0,
					);
			}

			return cache[i];
		}
	}

	function computeImpulseResponse(b, a, cache, causal) {
		const result = Array(sampleRate / 2)
			.fill(0)
			.map((_, i) => getImpulseResponseAt(b, a, i, cache, causal));

		if (causal) {
			return [...Array(sampleRate / 2).fill(0), ...result];
		} else {
			return [...result.reverse(), ...Array(sampleRate / 2).fill(0)];
		}
	}

	$: impulseResponse = computeImpulseResponse(b, a, [], causal);
	$: output = input.map((_, i) => getOutputAt(input, b, a, i, [], causal));
	$: outputFormularTime =
		"y[n] = " +
		([
			...b
				.map((c, i) =>
					c == 0
						? false
						: causal
							? i > 0
								? `${signedFormatter.format(c)} ⋅ x[n-${i}]`
								: `${formatter.format(c)} ⋅ x[n]`
							: `${(i > 0 ? signedFormatter : formatter).format(c)} ⋅ x[n+${i + 1}]`,
				)
				.filter((t) => t != false),
			...a
				.map((c, i) =>
					c == 0
						? false
						: causal
							? i > 0
								? `${signedFormatter.format(c)} ⋅ y[n-${i + 1}]`
								: `${formatter.format(c)} ⋅ y[n-1]`
							: `${(i > 0 ? signedFormatter : formatter).format(c)} ⋅ y[n+${i + 1}]`,
				)
				.filter((t) => t != false),
		].join(" + ") || 0);

	$: outputFormularFreq =
		"Y(z) = " +
		[
			[
				...b
					.map((c, i) =>
						c == 0
							? false
							: causal
								? i > 0
									? `${signedFormatter.format(c)}⋅z^(-${i}) ⋅ X(z)`
									: `${formatter.format(c)} ⋅ X(z)`
								: i > 0
									? `${signedFormatter.format(c)}⋅z^(${i + 1}) ⋅ X(z)`
									: `${formatter.format(c)}⋅z^${i + 1} ⋅ X(z)`,
					)
					.filter((t) => t != false),
			].join(" ") || 0,
			[
				...a
					.map((c, i) =>
						c == 0
							? false
							: causal
								? `${(i > 0 ? signedFormatter : formatter).format(c)}⋅z^(-${i + 1}) ⋅ Y(z)`
								: `${(i > 0 ? signedFormatter : formatter).format(c)}⋅z^${i + 1} ⋅ Y(z)`,
					)
					.filter((t) => t != false),
			].join(" ") || 0,
		]
			.filter((v, i) => v != 0)
			.join(" + ");

	$: outputFormularTransfer =
		"Y(z)/X(z) = (" +
		([
			...b
				.map((c, i) =>
					c == 0
						? false
						: causal
							? i > 0
								? `${signedFormatter.format(c)}⋅z^(-${i})`
								: `${formatter.format(c)}`
							: i > 0
								? `${signedFormatter.format(c)}⋅z^${i + 1}`
								: `${formatter.format(c)}⋅z^${i + 1}`,
				)
				.filter((t) => t != false),
		].join(" ") || 0) +
		") / (1" +
		(
			[
				...a
					.map((c, i) =>
						c == 0
							? false
							: causal
								? `${signedFormatter.format(-c)}⋅z^(-${i + 1})`
								: `${signedFormatter.format(-c)}⋅z^${i + 1}`,
					)
					.filter((t) => t != false),
			].join(" ") || ""
		).trim() +
		")";

	let selectedOutput = 0;

	let printPolar = false;

	$: outputs = [
		outputFormularTime,
		outputFormularFreq,
		outputFormularTransfer,
	];

	function updateCausality() {
		const [p, z] = toPoleZero(b, a, causal);
		poles = p;
		zeros = z;
		selectedPoleZero = null;
	}

	function setOrderA(o) {
		const orderA = Math.min(Math.max(o, minOrderA), maxOrderA);

		a = Array(orderA)
			.fill(0)
			.map((z, i) => a[i] || z);
		const [p, z] = toPoleZero(b, a, causal);
		poles = p;
		zeros = z;
		selectedPoleZero = null;
	}

	function setOrderB(o) {
		const orderB = Math.min(Math.max(o, minOrderB), maxOrderB) + 1;

		b = Array(orderB)
			.fill(0)
			.map((z, i) => b[i] || z);
		const [p, z] = toPoleZero(b, a, causal);
		poles = p;
		zeros = z;
		selectedPoleZero = null;
	}

	function inputOrderA(evt) {
		setOrderA(parseInt(evt.currentTarget.value, 10));
	}

	function inputOrderB(evt) {
		setOrderB(parseInt(evt.currentTarget.value, 10));
	}

	function resetInvalidB(evt) {
		const index = parseInt(
			evt.currentTarget.getAttribute("data-index"),
			10,
		);
		evt.currentTarget.classList.remove("error");
		evt.currentTarget.value = b[index];
	}

	function inputParamB(evt, scaleAll = false) {
		const v = parseFloat(evt.currentTarget.value);
		const index = parseInt(
			evt.currentTarget.getAttribute("data-index"),
			10,
		);

		if (isNaN(v)) {
			evt.currentTarget.classList.add("error");
			return;
		} else {
			evt.currentTarget.classList.remove("error");
		}

		const lock = evt.currentTarget.parentNode.querySelector(".lock-check");

		scaleAll = lock && lock.checked;

		if (scaleAll && b[index] != 0 && v != 0) {
			const scale = v / b[index];

			for (var i = b.length - 1; i >= 0; i--) {
				const newV = b[i] * scale;

				if (i !== index) {
					b[i] = newV;
				}
			}
		}

		if (b[index] !== v) {
			b[index] = v;
		}

		const [p, z] = toPoleZero(b, a, causal);
		poles = p;
		zeros = z;
		selectedPoleZero = null;
	}

	function inputParamA(evt, scaleAll = false) {
		const v = parseFloat(evt.currentTarget.value);
		const index = parseInt(
			evt.currentTarget.getAttribute("data-index"),
			10,
		);

		if (isNaN(v)) {
			evt.currentTarget.classList.add("error");
			return;
		} else {
			evt.currentTarget.classList.remove("error");
		}

		if (scaleAll && a[index] !== 0) {
			const scale = v / b[index];

			a = a.map((old) => old * scale);
		} else {
			a[index] = v;
		}

		const [p, z] = toPoleZero(b, a, causal);
		poles = p;
		zeros = z;
		selectedPoleZero = null;
	}

	function isValidParamB(a, index) {
		return (
			typeof b[index] == "number" && !isNaN(b[index]) && true // (index > 0 || b[index] != 0 || !b.some((x) => x != 0))
		);
	}

	function isValidParamA(b, index) {
		return typeof a[index] == "number" && !isNaN(b[index]);
	}

	$: document.body.classList[dragState != null ? "add" : "remove"](
		"dragging",
	);
</script>

<svelte:window on:mousemove={dragUpdate} on:mouseup|capture={dragStop} />
<article>
	<a href="https://tools.laszlokorte.de" title="More educational tools"
		>More educational tools</a
	>

	<h1>Digital Filter</h1>

	<p style="text-align: center;">
		Examples:
		{#each Object.keys(examples) as x}
			<button class="pill" on:click={() => loadExample(x)}>{x}</button>
		{/each}
	</p>

	<details>
		<summary>Introduction</summary>

		<p>
			Below you can see a linear time invariant system. That is a system
			that takes a time discrete signal as an input (on the left side) and
			produces an resulting signal as output (on the right side). It does
			so in a very constrained way: the input signal can only be split,
			delayed, scaled and added back together.
		</p>

		<p>
			The system is made up of two parts: Feed forward part (on the left
			side) create a number of copies of the input and delays each of them
			by one timestep. These delayed copies are then scaled individually
			and then summeed back together.
		</p>
		<p>
			Then in the Feed back part (on the right side) the previous result
			is split and delayed again but then <strong>recursively</strong> summed
			back together with itself. This creates a feedback loop and allowes to
			scale the signal exponentially.
		</p>
		<p>
			On the left side you can choose from various input signal to see the
			resulting output on the right side. Below the output signal graph
			you can see the recursive formular describing the system behaviour.
			This formular can be transformed into a rational function (a
			fraction with a polynomial in both the numerator and the
			denominator).
		</p>
		<p>
			The values for which the polynomials evaluate to zero are called
			zeros (for the numerator) and poles (for the denominator). The poles
			and zeroes are an alternative representation for the full system.
			The poles represent frequencies that amplified indefinitly when feed
			into the system, or in other words inputs for which the system
			behaviour is not stable.
		</p>
		<p>
			The poles (x) and zeros (o) are marked in the Pole/Zero plot. For
			the system to be stable it is required that all poles are located
			inside the unit circle. That is because all points outside the unit
			circle represent frequencies with no exponential growth. A system
			with poles outside the unit circle would amplify even constant or
			decaying input signals to produce an exponentially growing output
			signal. A system with poles inside the unit circle would only
			produce unbounded outputs for unbounded inputs (which are expected
			to not be given). The area beyong the outer most pole is called the
			regin of convergence. That is the set of a frequencies for which the
			system output is limited.
		</p>
	</details>

	<div class="system">
		<div class="causal-settings">
			<label
				><input
					type="radio"
					value={true}
					bind:group={causal}
					on:change={updateCausality}
				/> Causal</label
			>
			<label
				><input
					type="radio"
					value={false}
					bind:group={causal}
					on:change={updateCausality}
				/> Anti-Causal</label
			>
		</div>

		<fieldset class="feed-forward-settings">
			<legend> Feed Forward </legend>
			<dl>
				<dt><label for="orderB">Delay count</label></dt>
				<dd>
					<input
						id="orderB"
						size="4"
						step="1"
						min={minOrderB}
						max={maxOrderB}
						type="number"
						value={b.length - 1}
						on:input={inputOrderB}
					/>
				</dd>
				<dd class="full">
					<input
						type="range"
						step="1"
						min={minOrderB}
						max={maxOrderB}
						value={b.length - 1}
						on:input={inputOrderB}
					/>
				</dd>
			</dl>
		</fieldset>

		<fieldset class="feed-back-settings">
			<legend> Feed Back </legend>
			<dl>
				<dt><label for="orderA">Delay Count</label></dt>
				<dd>
					<input
						id="orderA"
						size="4"
						step="1"
						min={minOrderA}
						max={maxOrderA}
						type="number"
						value={a.length}
						on:input={inputOrderA}
					/>
				</dd>
				<dd class="full">
					<input
						type="range"
						step="1"
						min={minOrderA}
						max={maxOrderA}
						value={a.length}
						on:input={inputOrderA}
					/>
				</dd>
			</dl>
		</fieldset>

		<h2 class="system-titel">System</h2>

		<div class="in-signal">
			<h2>input signal x</h2>
			<svg class="signal-graph" viewBox="-50 -50 100 100">
				<line
					x1="-50"
					x2="50"
					y1="0"
					y2="0"
					stroke="#aaa"
					vector-effect="non-scaling-stroke"
				/>
				<line
					y1="-50"
					y2="50"
					x1="0"
					x2="0"
					stroke="#aaa"
					vector-effect="non-scaling-stroke"
				/>
				<path d="M50,0l-4,-3v6z" fill="#aaa" />
				<path d="M0,-50l-3,4h6z" fill="#aaa" />

				<text x="5" y="-42" font-size="7" fill="#777">X[n]</text>
				<text x="47" y="-5" font-size="7" fill="#777" text-anchor="end"
					>n</text
				>

				{#each input as v, i}
					<line
						y1="0"
						y2={-v * 25}
						x1={((i - input.length / 2) * 90) / input.length}
						x2={((i - input.length / 2) * 90) / input.length}
						stroke="#0af"
						vector-effect="non-scaling-stroke"
						stroke-width="2"
					/>
					<circle
						fill="#0af"
						r="1"
						cy={-v * 25}
						cx={((i - input.length / 2) * 90) / input.length}
					/>
				{/each}
			</svg>
			<select bind:value={inputName}>
				{#each Object.keys(inputs) as name}
					<option>{name}</option>
				{/each}
			</select>
		</div>

		<div class="out-signal">
			<h2>output signal y</h2>
			<svg class="signal-graph" viewBox="-50 -50 100 100">
				<line
					x1="-50"
					x2="50"
					y1="0"
					y2="0"
					stroke="#aaa"
					vector-effect="non-scaling-stroke"
				/>
				<line
					y1="-50"
					y2="50"
					x1="0"
					x2="0"
					stroke="#aaa"
					vector-effect="non-scaling-stroke"
				/>
				<path d="M50,0l-4,-3v6z" fill="#aaa" />
				<path d="M0,-50l-3,4h6z" fill="#aaa" />
				<text x="5" y="-42" font-size="7" fill="#777">Y[n]</text>
				<text x="47" y="-5" font-size="7" fill="#777" text-anchor="end"
					>n</text
				>

				{#each output as v, i}
					<line
						y1="0"
						y2={-v * 25}
						x1={((i - output.length / 2) * 90) / output.length}
						x2={((i - output.length / 2) * 90) / output.length}
						stroke="#0a6"
						vector-effect="non-scaling-stroke"
						stroke-width="2"
					/>
					<circle
						fill="#0a6"
						r="1"
						cy={-v * 25}
						cx={((i - output.length / 2) * 90) / output.length}
					/>
				{/each}
			</svg>

			<h2>Output Function</h2>

			<div class="checkbox-row">
				<label
					><input
						class="checkbox-hidden"
						type="radio"
						bind:group={selectedOutput}
						value={0}
					/> <span class="checkbox-label">Time</span></label
				>
				<label
					><input
						class="checkbox-hidden"
						type="radio"
						bind:group={selectedOutput}
						value={1}
					/> <span class="checkbox-label">Freq</span></label
				>
				<label
					><input
						class="checkbox-hidden"
						type="radio"
						bind:group={selectedOutput}
						value={2}
					/> <span class="checkbox-label">Transfere</span></label
				>
			</div>

			<textarea readonly class="formular"
				>{outputs[selectedOutput]}</textarea
			>

			<hr />
			<h2>Pole/Zero Plot</h2>

			<div class="checkbox-row">
				<label
					><input
						class="checkbox-hidden"
						type="radio"
						bind:group={printPolar}
						value={false}
					/> <span class="checkbox-label">Cartesian</span></label
				>
				<label
					><input
						class="checkbox-hidden"
						type="radio"
						bind:group={printPolar}
						value={true}
					/> <span class="checkbox-label">Polar</span></label
				>
			</div>

			{#if unsolvable}
				<div class="warning">
					The Amplificiation a<sub>0</sub> must not be zero. Otherwise
					the function can not be factored into a pole/zero representation.
				</div>
			{:else}
				<div>
					<div>
						<svg
							bind:this={pzSvg}
							class="signal-graph"
							viewBox="-50 -50 100 100"
							on:mousedown={() => {
								selectedPoleZero = null;
							}}
						>
							<line
								x1="-50"
								x2="50"
								y1="0"
								y2="0"
								stroke="#aaa"
								vector-effect="non-scaling-stroke"
							/>
							<line
								y1="-50"
								y2="50"
								x1="0"
								x2="0"
								stroke="#aaa"
								vector-effect="non-scaling-stroke"
							/>
							<path d="M50,0l-4,-3v6z" fill="#aaa" />
							<path d="M0,-50l-3,4h6z" fill="#aaa" />

							<text x="5" y="-42" font-size="7" fill="#777"
								>Im</text
							>
							<text
								x="47"
								y="-5"
								font-size="7"
								fill="#777"
								text-anchor="end">Re</text
							>

							<circle
								cx="0"
								cy="0"
								r="20"
								stroke="#aaa"
								fill="none"
								vector-effect="non-scaling-stroke"
							/>

							{#if causal}
								<path
									fill-rule="evenodd"
									d="M-50,-50L50,-50L50,50L-50,50z M{-rocRadiusMin *
										20},0a{rocRadiusMin *
										20} {rocRadiusMin * 20} 0 1 1 0 1z"
									fill="lime"
									fill-opacity="0.1"
								/>
							{:else}
								<circle
									cx="0"
									cy="0"
									r={(causal ? rocRadiusMin : rocRadiusMax) *
										20}
									fill-opacity="0.1"
									fill="lime"
								/>
							{/if}

							<circle
								cx="0"
								cy="0"
								r={(causal ? rocRadiusMin : rocRadiusMax) * 20}
								stroke-width="1"
								stroke="green"
								fill="none"
								stroke-dasharray="3 3"
								vector-effect="non-scaling-stroke"
							/>

							{#if zeros.length}
								{#each zeros as [zx, zy], i}
									{#if isFinite(zx) && isFinite(zy)}
										<circle
											cx={zx * 20}
											cy={-zy * 20}
											r="2"
											fill="none"
											stroke="red"
											vector-effect="non-scaling-stroke"
											class:selected-zero={selectedPoleZero ==
												`z-${i}`}
										>
											<title>Root</title>
										</circle>

										<circle
											cx={zx * 20}
											cy={-zy * 20}
											r="3"
											fill="none"
											stroke="none"
											vector-effect="non-scaling-stroke"
											pointer-events="all"
											on:mousedown={(e) => {
												e.stopPropagation();
											}}
											on:click={(e) => {
												e.stopPropagation();
												selectedPoleZero = `z-${i}`;
											}}
										>
										</circle>
									{/if}
								{/each}
							{/if}
							{#if poles.length}
								{#each poles as [px, py], i}
									{#if isFinite(px) && isFinite(py)}
										<path
											d="M{svgNumber.format(
												px * 20,
											)}, {svgNumber.format(
												-py * 20,
											)}m-2,-2l4,4m-4,0l4,-4"
											vector-effect="non-scaling-stroke"
											stroke="blue"
											class:selected-pole={selectedPoleZero ==
												`p-${i}`}
										>
											<title>Pole</title>
										</path>

										<circle
											cx={px * 20}
											cy={-py * 20}
											r="3"
											fill="none"
											stroke="none"
											vector-effect="non-scaling-stroke"
											pointer-events="all"
											on:mousedown={(e) => {
												e.stopPropagation();
											}}
											on:click={(e) => {
												e.stopPropagation();
												selectedPoleZero = `p-${i}`;
											}}
										>
										</circle>
									{/if}
								{/each}
							{/if}

							{#if selectedPoleZeroPos}
								{#if isFinite(selectedPoleZeroPos[0]) && isFinite(selectedPoleZeroPos[1])}
									<circle
										cx={selectedPoleZeroPos[0] * 20}
										cy={-selectedPoleZeroPos[1] * 20}
										r="5"
										fill="none"
										stroke="none"
										data-polezero={selectedPoleZero}
										on:mousedown={dragStart}
										vector-effect="non-scaling-stroke"
										cursor="move"
										pointer-events="all"
									>
									</circle>
								{/if}
							{/if}
						</svg>

						<div class="caption">
							{#if stable}
								Stable
							{:else}
								Unstable
							{/if}
						</div>
					</div>

					<div>
						<div>
							<h3>Gain</h3>
							<ul>
								<li>{b[0]}</li>
							</ul>
							<h3>Poles</h3>
							<select size="3" bind:value={selectedPoleZero}>
								{#each poles as p, i}
									<option value="p-{i}"
										>{formatComplex(p, printPolar)}</option
									>
								{/each}
							</select>

							<h3>Zeros</h3>
							<select size="3" bind:value={selectedPoleZero}>
								{#each zeros as z, i}
									<option value="z-{i}"
										>{formatComplex(z, printPolar)}</option
									>
								{/each}
							</select>
						</div>

						<div>
							{#if selectedPoleZero}
								<h3>Selected</h3>
								<div
									style="display: flex;gap: 0.5em; align-items: center;"
								>
									<input
										data-i="0"
										data-polezero={selectedPoleZero}
										on:input={updatePole}
										disabled={!isFinite(
											selectedPoleZeroPos[0],
										)}
										value={isFinite(selectedPoleZeroPos[0])
											? formatter.format(
													selectedPoleZeroPos[0],
												)
											: selectedPoleZeroPos[0]}
										type={isFinite(selectedPoleZeroPos[0])
											? "number"
											: "text"}
										min="-10"
										max="10"
										step="0.1"
										style="width: 50%; flex-shrink: 1;"
									/>
									+
									<input
										data-i="1"
										data-polezero={selectedPoleZero}
										disabled={!isFinite(
											selectedPoleZeroPos[1],
										)}
										on:input={updatePole}
										value={isFinite(selectedPoleZeroPos[1])
											? formatter.format(
													selectedPoleZeroPos[1],
												)
											: selectedPoleZeroPos[1]}
										type={isFinite(selectedPoleZeroPos[1])
											? "number"
											: "text"}
										min="-10"
										max="10"
										step="0.1"
										style="width: 1fr; width: 50%; flex-shrink: 1;"
									/>
									i
								</div>
							{/if}
						</div>
					</div>
				</div>
			{/if}
		</div>

		<div class="feed-forward stack">
			{#if !causal}
				<div class="stack-box">
					<Mesh
						delay={causal ? 1 : -1}
						first={true}
						last={b.length < 1}
					/>
				</div>
			{/if}
			{#each b as v, i}
				<div class="stack-box">
					<Mesh
						delay={causal ? 1 : -1}
						first={i == 0 && causal}
						last={i >= b.length - 1}
					/>
					<div class="weight-field">
						<div class="horizontal">
							<label for="b_{i}">Amplify</label>
							{#if i == 0}
								<input
									id="lock"
									class="lock-check"
									type="checkbox"
								/>
								<label
									title="Lock Ratio"
									for="lock"
									class="lock"
								>
									<Link />
								</label>
							{/if}
						</div>
						<input
							lang="en"
							class:invalid={!isValidParamB(b, i)}
							min="-10"
							max="10"
							step="0.01"
							class="weight-input"
							id="b_{i}"
							data-index={i}
							type="number"
							value={formatter.format(v)}
							size="3"
							on:input={inputParamB}
							on:blur={resetInvalidB}
						/>
					</div>
				</div>
			{/each}
		</div>
		<div class="feed-back stack">
			<div class="stack-box">
				<Mesh
					delay={causal ? 1 : -1}
					first={true}
					last={a.length < 1}
					back
					out
				/>
			</div>
			{#each a as v, i}
				<div class="stack-box">
					<Mesh
						delay={causal ? 1 : -1}
						first={false}
						last={i >= a.length - 1}
						out
						back
					/>

					<div class="weight-field">
						<label for="a_{i}">Amplify</label>
						<input
							lang="en"
							class:invalid={!isValidParamA(a, i)}
							min="-10"
							max="10"
							step="0.01"
							class="weight-input"
							id="a_{i}"
							data-index={i}
							type="number"
							value={formatter.format(v)}
							size="3"
							on:input={inputParamA}
						/>
					</div>
				</div>
			{/each}
		</div>
	</div>
</article>

<style>
	:global(body) {
		overflow-y: scroll;
	}

	:global(.error),
	.invalid {
		outline: 2px solid #a00;
		background: #fee;
	}

	:global(body.dragging) {
		cursor: move;
	}

	dl {
		display: grid;
		grid-template-columns: [full-start] max-content max-content [full-end];
		align-items: start;
		gap: 0.2em 1em;
		margin: 0;
		justify-content: center;
	}

	.full {
		grid-column: full;
	}

	input {
		margin: 0;
		padding: 0.3em;
	}

	dd,
	dt {
		margin: 0;
	}

	dt {
		display: flex;
		align-items: start;
		padding: 0.3em;
	}

	article {
		max-width: 60em;
		margin: auto;
	}

	.system {
		padding: 1em;
		display: grid;
		grid-template-columns:
			[in-start] minmax(8em, 3fr)
			[in-end] 1em [forward-start causality-start] minmax(14em, 4fr)
			[forward-end backward-start] minmax(14em, 4fr)
			[backward-end causality-end] 1em [out-start] minmax(8em, 3fr)
			[out-end];
		grid-template-rows: [causality-start] auto [causality-end settings-start side-start] auto [settings-end signal-start logic-start] auto [logic-end signal-end side-end];
		grid-auto-flow: dense;
		gap: 1em 0;
	}

	.causal-settings {
		grid-column: causality;
		grid-row: causality;
		padding: 0 1em;
		align-self: center;
		display: flex;
		gap: 1em;
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
		grid-row: logic;
		padding-top: 3em;
	}

	.feed-back {
		grid-area: backward;
		grid-row: logic;
		padding-top: 3em;
	}

	.stack {
		display: flex;
		flex-direction: column;
	}

	.stack-box {
		display: grid;
		grid-template-columns: [cover-start] 35fr [center-start] 30fr [center-end] 35fr [cover-end];
		grid-template-rows: [cover-start center-start] 35% [center-end] 65% [cover-end];
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
		margin-bottom: 0.2em;
	}

	.weight-input {
		width: 100%;
		margin: 0;
		min-width: 3em;
	}

	.signal-graph {
		display: block;
		background: #fafafa;
		margin: 0 0.1em;
		width: 100%;
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
		font-size: 0.9em;
		font-weight: bold;
		margin: 0;
	}

	h2 {
		font-size: 1.2em;
		font-weight: normal;
		margin: 0;
		text-align: center;
	}

	h1 {
		text-align: center;
	}

	hr {
		justify-self: stretch;
		width: 100%;
		border: none;
		border-top: 1px solid gray;
	}

	.formular {
		font-family: monospace;
		height: 5em;
		overflow: auto;
		resize: vertical;
	}

	ul {
		list-style: none;
		margin: 0;
		text-align: left;
		padding: 0;
	}

	.system-titel {
		grid-column: forward-start / backward-end;
		grid-row: logic-start;
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
		flex-shrink: 0;
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
		justify-content: center;
		margin: 0.2em 0;
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

	select[size] {
		width: 100%;
		font-size: inherit;
		font: inherit;
		padding: 0;
	}

	.selected-pole,
	.selected-zero {
		stroke-width: 3;
	}

	.horizontal {
		display: flex;
		justify-content: space-between;
	}
</style>
