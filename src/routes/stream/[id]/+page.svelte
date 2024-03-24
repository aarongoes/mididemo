<script>
	import { Peer } from 'peerjs';
	import { page } from '$app/stores';
	var otherPeerId;
	var lastPeerId = null;
	var peer = null; // own peer object
	var conn = null;
	let midiIn = [];
	let midiOut = [];
	let selectedInput;
	let selectedOutput;

	/**
	 * Create the Peer object for our end of the connection.
	 *
	 * Sets up callbacks that handle any events related to our
	 * peer object.
	 */
	function initializePeer() {
		// Create own peer object with connection to shared PeerJS server
		peer = new Peer(null, {
			debug: 2
		});

		peer.on('open', function (id) {
			// Workaround for peer.reconnect deleting previous id
			if (peer.id === null) {
				console.log('Received null id from peer open');
				peer.id = lastPeerId;
			} else {
				lastPeerId = peer.id;
			}

			console.log('ID: ' + peer.id);
			join();
		});
		peer.on('connection', function (c) {
			// Disallow incoming connections
			c.on('open', function () {
				c.send('Sender does not accept incoming connections');
				setTimeout(function () {
					c.close();
				}, 500);
			});
		});
		peer.on('disconnected', function () {
			// status.innerHTML = 'Connection lost. Please reconnect';
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
	 * Create the connection between the two Peers.
	 *
	 * Sets up callbacks that handle any events related to the
	 * connection and data received on it.
	 */
	function join() {
		// Close old connection
		if (conn) {
			conn.close();
		}

		// Create connection to destination peer specified in the input field
		conn = peer.connect($page.params.id, {
			reliable: true
		});

		conn.on('open', function () {
			console.log('Connected to: ' + conn.peer);
		});
		// Handle incoming data (messages only since this is the signal sender)
		conn.on('data', function (data) {
			console.log(data);
		});
		conn.on('close', function () {
			console.log('connection closed');
		});
	}

	function send(json) {
		if (conn && conn.open) {
			conn.send(JSON.stringify(json));
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

		if (!selectedInput) {
			selectedInput = midiIn[0];
		}
		if (!selectedOutput) {
			selectedOutput = midiOut[0];
		}

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
		if (event.target == selectedInput) {
			const NOTE_ON = 9;
			const NOTE_OFF = 8;

			console.log(event);
			const cmd = event.data[0] >> 4;
			const pitch = event.data[1];
			const velocity = event.data.length > 2 ? event.data[2] : 1;
			send({ pitch: pitch, velocity: velocity, command: cmd });
		}
		// MIDI commands we care about. See
		// http://webaudio.github.io/web-midi-api/#a-simple-monophonic-sine-wave-midi-synthesizer.

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
		// 		`🎧 from ${event.srcElement.name} note on: pitch:${pitch}, velocity: ${velocity}`
		// 	);

		// 	// One note can only be on at once.
		// 	// notesOn.set(pitch, timestamp);
		// }
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

	connect();
	initializePeer();
</script>

<!-- <input bind:value={otherPeerId}/>
<button on:click={join}/> -->
<button on:click={send} />
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
