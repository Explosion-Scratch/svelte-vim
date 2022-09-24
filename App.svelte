<script>
	import {onMount} from "svelte";
	
	const cursor = {
		line: 0,
		column: 0,
	}
	let command = "";
	let keyListeners = [];
	
	let content = `Sirloin aute leberkas occaecat.\nTurkey sirloin incididunt,\n\nchicken leberkas doner strip steak ribeye fugiat hamburger eiusmod ad non.\nChislic quis chuck dolore do swine ribeye pork chop.\nBiltong rump sausage ut magna.\n\n\tCapicola culpa consequat,\nmollit shank ad eiusmod flank turducken.\nEnim swine bacon porchetta pork burgdoggen shankle ham irure.`;
	
	
	$: rows = content.split("\n");
	
	let pressed = [];
	let sequences = [];
	
	//The time in ms that the user is allowed to press another key like a number key to have the command just pressed repeat x times
	const DEBOUNCE_TIME = 2000;
	let currentMode = "normal";
	onMount(() => {
		Object.assign(window, {mode, currentMode, pressed, sequences, boundCursor, testSequence, id, callCommand, repeatSequence, removeLine, key, insertAt, keyListeners, cursor})
		mode("normal");
		mode("normal", () => {
			key(">>", () => insertAt({line: cursor.line, column: 0, cb: (a, b) => b + a}, "\t"))
			key("<<", () => {
				let row = rows[cursor.line];
				if (row.startsWith("\t")){
					row = row.replace(/^\t/, "");
					cursor.column--;
					content = rows.map((i, idx) => idx === cursor.line ? row : i) .join("\n");
				}
			})
			key("j", () => (cursor.line++));
			key("k", () => (cursor.line--));
			key("h", () => (cursor.column--));
			key("l", () => (cursor.column++));
			key("dd", () => removeLine(cursor.line));
			key("i", () => mode("insert"))
			key(':', () => mode('command'))
		})
		mode("command", () => {
			key("Escape", () => mode('normal'))
			key((a) => {
				return !(a.includes('Escape') || a.includes(':'))
			}, ({pressed}) => {
				let thing = pressed?.slice(-1)?.[0];
				if (thing === 'Backspace'){
					command = command.slice(0, -1);
				}
				if (!thing || thing.length !== 1){return}
				command += pressed.slice(-1)[0];
			})
			key("Enter", () => {
				if (["wq", "x", "zz"].includes(command)){
					commandStatus = "Saved";
					if (command === "wq"){
						commandStatus = "Saved + Exited"
					}
				}
				setTimeout(() => (commandStatus = ""), 2000);
			})
		})
		mode("insert", () => {
			key('Backspace', () => deleteAt(cursor))
			key(':', () => mode('command'))
			key((a) => {
				return !(a.includes('Escape') || a.includes(':'))
			}, ({pressed}) => {
				const thing = pressed.slice(-1)[0];
				if (thing.length !== 1){return}
				insertAt(cursor, thing);
			})
			key("Escape", () => mode("normal"));
		})
		for (let i = 1; i <= 9; i++){
			key(i.toString(), () => {
				if (!sequences?.slice(-1)?.[0]?.sequence){return}
				repeatSequence(sequences.slice(-1)[0].sequence, i - 1)
				sequences = [];
			})
		}
		document.addEventListener("keydown", (e) => {
			console.log("User pressed key ", e.key);
			pressed.push(e.key);
			
			for (const {sequence, ...cb} of keyListeners){
				let works = testSequence(sequence, pressed);
				if (currentMode !== cb.mode){
						continue;
				}
				if (works){
					console.log("Running sequence, ", sequence, cb)
					cb.cb({pressed, automated: false, event: e, ...cb})
					pressed = [];
					let _id = id();
					sequences.push({sequence, id: _id});
					setTimeout(() => {
						sequences = sequences.filter(({id}) => id !== _id);
					}, DEBOUNCE_TIME)
				}
			}
		})
		document.addEventListener("keyup", () => {
				boundCursor();
		})
	})
	
	function mode(thing, cb){
		if (cb){
			let om = currentMode;
			currentMode = thing;
			cb({old: om, new: thing});
			currentMode = om;
		} else {
			currentMode = thing;
		}
	}
	function insertAt(loc, thing, move = {column: 1}){
		let new_rows = [...rows];
		new_rows[loc.line] = rows[loc.line].split("").map((i, idx) => idx === loc.column ? (loc.cb ? loc.cb(i, thing) : i + thing) : i).join("");
		cursor.column += move.column || 0;
		cursor.line += move.line || 0;
		content = new_rows.join("\n");
	}
	function deleteAt(loc){
		let new_rows = [...rows];
		new_rows[loc.line] = rows[loc.line].split("").map((i, idx) => idx === loc.column ? "" : i).join("");
		cursor.column--;
		content = new_rows.join("\n");
	}
	function boundCursor(){
		if (cursor.line >= rows.length){
			cursor.line = rows.length - 1;
		}
		if (cursor.line < 0){
			cursor.line = 0;
		}
		if (cursor.column < 0){
			cursor.column = 0;
		}
		if (cursor.column >= rows[cursor.line].length){
			cursor.column = rows[cursor.line].length - 1;
		}
	}
	function testSequence(sequence, pressed){
		if (sequence instanceof Function){
			return !!sequence(pressed);
		}
		if (sequence instanceof RegExp){
			return sequence.test(pressed.join(""));
		}
		if (typeof sequence === "string"){
			return pressed.join("").endsWith(sequence)/*.indexOf(sequence) !== -1 */;
		}
		if (Array.isArray(sequence)){
			return sequence.sort().toString() === pressed.sort().toString();
		}
	}
	async function callCommand(sequence){
		console.log(keyListeners, sequence)
		for (const s of keyListeners){
			if (!sequence?.split){
				return console.log(sequence)
			}
			if (testSequence(s.sequence, sequence.split("")) && s.mode === currentMode){
				await s.cb({pressed, automated: true, ...s});
			}
		}
	}
	function id(){
		return Math.random().toString(16).slice(2);
	}
	async function repeatSequence(sequence, times){
			//It's a number, no recursion
			if (parseInt(sequence)){return}
			console.log("Repeating sequence %o x%o times", sequence, times)
			for (let i = 0; i < times; i++){
				await callCommand(sequence);
			}
	}
	function removeLine(line){
		content = [...rows].filter((_, idx) => idx !== line).join("\n");
	}
	function key(code, cb, ...other){
		keyListeners.push({cb, mode: currentMode, ...other, sequence: code});
	}
</script>
<div class="vim">
{#each rows as row,row_num}
	<span class="gutter">{row_num}</span
	>{#if row.length === 0 && cursor.line === row_num}
		&nbsp;<span class="cursor">&nbsp;</span>
	{/if}
	{#each row as column,col_num}
		{#if col_num === cursor.column && row_num === cursor.line && currentMode !== 'command'}
			<span class="cursor">{column}</span>
		{:else}
			{column}
		{/if}
	{/each}
	<br>
{/each}
<br><br>
<span class="toolbar">
	{#if currentMode !== "command"}
		<span class="right">{cursor.line}:{cursor.column}</span>
		<span class="left">{new Date().toString().split(" ").slice(1, 3).join(" ")}&#9;--{currentMode.toUpperCase()}--</span>
	{:else}
		<span class="right">:{#if command.length}{command.slice(0, -1)}<span class="cursor">{command.slice(-1)}</span>{:else}<span class="cursor">&nbsp;</span>{/if}
		</span>
		<span class="left">Enter a command</span>
	{/if}
</span>
</div>
<style>
	:root {
		--background: #666;
		--text: #eee;
		--cursor-bg: var(--text);
		--cursor-color: var(--background);
		--toolbar-bg: var(--text);
		--toolbar-color: var(--background);
	}
	.vim {
		color: var(--text);
		background: var(--background);
		font-family: 'Courier New', monospace;
		white-space: pre;
	}
	.cursor {
		background: var(--cursor-bg);
		color: var(--cursor-color);
	}
	.toolbar {
		display: flex;
		justify-content: space-between;
		padding: 0 2ch;
		background: var(--toolbar-bg);
		color: var(--toolbar-color);
	}
</style>