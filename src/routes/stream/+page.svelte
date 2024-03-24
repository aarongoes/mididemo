<script>
	import { Qr } from '@sveltevk/qr';
	import { Peer } from 'peerjs';
	import { onMount } from 'svelte';
	import { page } from '$app/stores';
	import { json } from '@sveltejs/kit';
	let url = '';
	var lastPeerId = null;
	var peer = null; // Own peer object
	var peerId = null;
	var conn = null;
	let midiIn = [];
	let midiOut = [];
	let selectedInput;
	let selectedOutput;

	function initializePeer() {
		// Create own peer object with connection to shared PeerJS server
		peer = new Peer({
			debug: 2
		});

		peer.on('open', function (id) {
			url = $page.url.origin + '/stream/' + id;
		});
		peer.on('connection', function (c) {
			// Allow only a single connection
			if (conn && conn.open) {
				c.on('open', function () {
					c.send('Already connected to another client');
					setTimeout(function () {
						c.close();
					}, 500);
				});
				return;
			}

			conn = c;
			console.log('Connected to: ' + conn.peer);
			ready();
		});
		peer.on('disconnected', function () {
			console.log('Connection lost. Please reconnect');

			// Workaround for peer.reconnect deleting previous id
			peer.id = lastPeerId;
			peer._lastServerId = lastPeerId;
			peer.reconnect();
		});
		peer.on('close', function () {
			conn = null;
			console.log('Connection destroyed');
		});
		peer.on('error', function (err) {
			console.log(err);
			alert('' + err);
		});
	}

	/**
	 * Triggered once a connection has been achieved.
	 * Defines callbacks to handle incoming data and connection events.
	 */
	function ready() {
		conn.on('data', function (data) {
			console.log(data);
			sendMidiCommand(JSON.parse(data));
		});
		conn.on('close', function () {
			conn = null;
		});
	}

	function send() {
		if (conn && conn.open) {
			conn.send('test');
			console.log(' signal sent');
			// addMessage(cueString + sigName);
		} else {
			console.log('Connection is closed');
		}
	}

	function connect() {
		navigator.requestMIDIAccess().then(
			(midi) => midiReady(midi),
			(err) => console.log('Something went wrong', err)
		);
	}

	function midiReady(midi) {
		// Also react to device changes.
		midi.addEventListener('statechange', (event) => initDevices(event.target));
		initDevices(midi); // see the next section!
	}

	function initDevices(midi) {
		// Reset.
		midiIn = [];
		midiOut = [];

		// MIDI devices that send you data.
		const inputs = midi.inputs.values();
		for (let input = inputs.next(); input && !input.done; input = inputs.next()) {
			midiIn.push(input.value);
		}

		// MIDI devices that you send data to.
		const outputs = midi.outputs.values();
		for (let output = outputs.next(); output && !output.done; output = outputs.next()) {
			midiOut.push(output.value);
		}

		selectedInput = midiIn[0];
		selectedOutput = midiOut[0];
		console.log(midiIn);
		console.log(midiOut);
		startListening();
	}

	// Start listening to MIDI messages.
	function startListening() {
		for (const input of midiIn) {
			input.addEventListener('midimessage', midiMessageReceived);
		}
	}

	function midiMessageReceived(event) {
		// MIDI commands we care about. See
		// http://webaudio.github.io/web-midi-api/#a-simple-monophonic-sine-wave-midi-synthesizer.
		const NOTE_ON = 9;
		const NOTE_OFF = 8;

		const cmd = event.data[0] >> 4;
		const pitch = event.data[1];
		const velocity = event.data.length > 2 ? event.data[2] : 1;

		// You can use the timestamp to figure out the duration of each note.
		// const timestamp = Date.now();

		// // Note that not all MIDI controllers send a separate NOTE_OFF command for every NOTE_ON.
		// if (cmd === NOTE_OFF || (cmd === NOTE_ON && velocity === 0)) {
		// 	console.log(
		// 		`🎧 from ${event.srcElement.name} note off: pitch:${pitch}, velocity: ${velocity}`
		// 	);

		// 	// Complete the note!
		// 	// const note = notesOn.get(pitch);
		// 	if (note) {
		// 		console.log(`🎵 pitch:${pitch}, duration:${timestamp - note} ms.`);
		// 		// notesOn.delete(pitch);
		// 	}
		// } else if (cmd === NOTE_ON) {
		// 	console.log(
		// 		`🎧 from ${event.srcElement.name} note off: pitch:${pitch}, velocity: {velocity}`
		// 	);

		// 	// One note can only be on at once.
		// 	// notesOn.set(pitch, timestamp);
		// }
	}
	function sendMidiCommand(command) {
		const NOTE_ON = 9;
		const NOTE_OFF = 8;
		const NOTE_ON_BINARY = 0x90;
		const NOTE_OFF_BINARY = 0x80;

		const device = selectedOutput;

		if (command.command == NOTE_ON) {
			device.send([NOTE_ON_BINARY, command.pitch, command.velocity]);
		} else {
			device.send([NOTE_OFF_BINARY, command.pitch, command.velocity]);
		}
	}
	function sendMidiMessage(pitch, velocity, duration) {
		const NOTE_ON = 0x90;
		const NOTE_OFF = 0x80;

		const device = selectedOutput;
		const msgOn = [NOTE_ON, pitch, velocity];
		const msgOff = [NOTE_OFF, pitch, velocity];

		// First send the note on;
		device.send(msgOn);

		// Then send the note off. You can send this separately if you want
		// (i.e. when the button is released)
		device.send(msgOff, Date.now() + duration);
	}

	function copyUrl(){
		navigator.clipboard.writeText(url);
	}

	initializePeer();
	connect();
</script>

<h1>Stream</h1>
<p>Share this link to stream your Midi inputs: <a href={url}>{url}</a></p>
<button on:click={copyUrl}>copy</button>
<Qr text={url} qrSize={256} />
<div id="qrcode" />
<button on:click={send}>send</button>
<select bind:value={selectedInput}>
	{#each midiIn as midi}
		<option value={midi}>
			{midi.name}
		</option>
	{/each}
</select>
<select bind:value={selectedOutput}>
	{#each midiOut as midi}
		<option value={midi}>
			{midi.name}
		</option>
	{/each}
</select>
