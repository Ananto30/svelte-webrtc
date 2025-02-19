<script lang="ts">
	import { onDestroy, onMount } from 'svelte';
	import { slide } from 'svelte/transition';

	let localVideo: HTMLVideoElement;
	let remoteVideo: HTMLVideoElement;
	let pollingInterval: number;
	let pc: RTCPeerConnection;
	let iceCandidates: Array<{ candidate: RTCIceCandidateInit }> = [];
	let iceCandidateRequestInitiated = false;
	let myEvents: any[] = [];

	let client_id: string;
	let room_id: string;
	let error: string;
	let message: string = 'Enter your name and join the room to start video call';

	const configuration: RTCConfiguration = {
		iceServers: [
			{
				urls: 'stun:stun.l.google.com:19302'
			}
		]
	};

	const serverUrl = 'https://vercel-webrtc.vercel.app/api';
	// const serverUrl = 'http://localhost:3000/api';

	onMount(() => {
		const hash = window.location.hash.substring(1);
		if (hash) {
			room_id = hash;
		} else {
			room_id = Math.random().toString(36).substr(2, 9);
			window.location.hash = room_id;
		}

		// Retrieve client_id from local storage
		client_id = localStorage.getItem('client_id') || '';
	});

	function isMobile() {
		return /Android|webOS|iPhone|iPad|iPod|BlackBerry|IEMobile|Opera Mini/i.test(
			navigator.userAgent
		);
	}

	function joinRoom() {
		if (!room_id) {
			error = 'Room ID is required';
			return;
		}
		if (!client_id) {
			error = 'Name is required';
			return;
		}

		// Save client_id to local storage
		localStorage.setItem('client_id', client_id);

		error = '';
		message = 'Joining room...';

		fetch(`${serverUrl}/join`, {
			method: 'POST',
			headers: {
				'Content-Type': 'application/json'
			},
			body: JSON.stringify({
				room_id: room_id,
				client_id: client_id
			})
		})
			.then((response) => response.json())
			.then((data) => {
				const members: string[] = data.members;
				console.log('MEMBERS', members);
				// If we are the second user to connect to the room we will be creating the offer
				if (members.length > 2) {
					message = 'This room is full. Please try another room.';
					return;
				}

				const isOfferer = members.length === 2;
				if (!isOfferer) {
					message = 'Wait for other person to connect';
				} else {
					message = 'Other person is already here. Connecting...';
				}
				startWebRTC(isOfferer);
				startPolling();

				startLocalVideo();
			})
			.catch((error) => {
				console.error('Error joining room:', error);
				message = 'Error joining room. Please try again.';
			});
	}

	function leaveRoom() {
		fetch(`${serverUrl}/leave`, {
			method: 'POST',
			headers: {
				'Content-Type': 'application/json'
			},
			body: JSON.stringify({
				room_id: room_id,
				client_id: client_id
			})
		})
			.then((response) => response.json())
			.then((data) => {
				console.log('Left room:', data);
				window.location.href = '/';
			})
			.catch((error) => console.error('Error leaving room:', error));

		if (pollingInterval) {
			clearInterval(pollingInterval);
		}

		if (pc) {
			pc.close();
		}

		localVideo.srcObject = null;
		remoteVideo.srcObject = null;

		message = 'You have left the room';
	}

	function onSuccess() {}

	function onError(error: any) {
		console.error(error);
	}

	// Send signaling data via HTTP
	function sendMessages(messages: any[]) {
		fetch(`${serverUrl}/event`, {
			method: 'POST',
			headers: {
				'Content-Type': 'application/json'
			},
			body: JSON.stringify({ room_id: room_id, messages })
		}).catch((error) => console.error('Error sending messages:', error));
	}

	function gatherAndSendIceCandidates() {
		if (iceCandidates.length > 0 && !iceCandidateRequestInitiated) {
			iceCandidateRequestInitiated = true;
			setTimeout(() => {
				sendMessages(iceCandidates);
				iceCandidates = [];
				iceCandidateRequestInitiated = false;
			}, 2000);
		}
	}

	function addToMyEvents(event: any) {
		myEvents.push(JSON.parse(JSON.stringify(event)));
	}

	function startWebRTC(isOfferer: boolean) {
		navigator.mediaDevices
			.getUserMedia({
				audio: {
					// echoCancellation: true,
					// noiseSuppression: true,
					sampleRate: 44100
				},
				video: {
					width: { ideal: 1280, max: 1920 },
					height: { ideal: 720, max: 1080 },
					facingMode: 'user' // Use 'user' for front camera, 'environment' for back camera
				}
			})
			.then((stream) => {
				// Initialize RTCPeerConnection before adding tracks
				pc = new RTCPeerConnection(configuration);

				// // Display your local video in #localVideo element
				// localVideo.srcObject = stream;
				// localVideo.play();

				// Add your stream to be sent to the connecting peer
				stream.getTracks().forEach((track) => pc.addTrack(track, stream));

				// 'onicecandidate' notifies us whenever an ICE agent needs to deliver a
				// message to the other peer through the signaling server
				pc.onicecandidate = (event) => {
					// when there are onicecandidate events, they come as bunches, so we can gather them and send them together to reduce http requests
					if (event.candidate) {
						// sendMessage({ candidate: event.candidate });
						iceCandidates.push({ candidate: event.candidate });
						addToMyEvents({ candidate: event.candidate });
						gatherAndSendIceCandidates();
					}
				};

				// If user is offerer let the 'negotiationneeded' event create the offer
				if (isOfferer) {
					pc.onnegotiationneeded = () => {
						pc.createOffer().then(localDescCreated).catch(onError);
					};
				}

				// When a remote stream arrives display it in the #remoteVideo element
				pc.ontrack = (event) => {
					const stream = event.streams[0];
					if (!remoteVideo.srcObject || remoteVideo.srcObject.id !== stream.id) {
						remoteVideo.srcObject = stream;
						remoteVideo.play(); // Ensure the remote video starts playing
						message = 'You are both here! Enjoy your video call. 🎉';

						// stop polling
						if (pollingInterval) {
							clearInterval(pollingInterval);
						}
					}
				};

				// Listen for ICE connection state changes
				pc.oniceconnectionstatechange = () => {
					if (
						pc.iceConnectionState === 'disconnected' ||
						pc.iceConnectionState === 'failed' ||
						pc.iceConnectionState === 'closed'
					) {
						remoteVideo.srcObject = null;
						remoteVideo.style.backgroundColor = 'black';
						message = 'Connection lost. Please wait for the other person to reconnect.';

						// start polling again
						startPolling();
					}
				};
			})
			.catch((error) => {
				message =
					'Error accessing media devices. Without camera permission video session cannot be started.';
				// Set the video elements' background color to black if no stream is available
				localVideo.style.backgroundColor = 'black';
				remoteVideo.style.backgroundColor = 'black';
			});
	}

	function localDescCreated(desc: RTCSessionDescriptionInit) {
		pc.setLocalDescription(desc)
			.then(() => {
				sendMessages([{ sdp: pc.localDescription }]);
				addToMyEvents({ sdp: pc.localDescription });
			})
			.catch(onError);
	}

	function startPolling() {
		pollingInterval = setInterval(() => {
			fetch(`${serverUrl}/events?room_id=${room_id}`)
				.then((response) => response.json())
				.then((messages) => {
					messages.events.forEach((message: any) => {
						if (message.sdp) {
							if (myEvents.some((event) => event.sdp?.type === message.sdp.type)) {
								return;
							}

							// This is called after receiving an offer or answer from another peer
							const remoteDesc = new RTCSessionDescription(message.sdp);
							if (pc.signalingState === 'stable' || pc.signalingState === 'have-local-offer') {
								pc.setRemoteDescription(remoteDesc)
									.then(() => {
										// When receiving an offer lets answer it
										if (remoteDesc.type === 'offer') {
											pc.createAnswer().then(localDescCreated).catch(onError);
										}
									})
									.catch(onError);
							} else {
								console.warn('Cannot set remote description in state:', pc.signalingState);
							}
						} else if (message.candidate) {
							if (
								myEvents.some((event) => event.candidate?.candidate === message.candidate.candidate)
							) {
								return;
							}

							// Add the new ICE candidate to our connections remote description
							pc.addIceCandidate(new RTCIceCandidate(message.candidate))
								.then(onSuccess)
								.catch(onError);
						}
					});
				})
				.catch((error) => console.error('Error polling messages:', error));
		}, 5000); // Poll every second
	}

	let copyMessage = '';
	function copyToClipboard() {
		const url = window.location.href;
		navigator.clipboard
			.writeText(url)
			.then(() => {
				copyMessage = 'URL copied to clipboard!';
				setTimeout(() => {
					copyMessage = '';
				}, 2000);
			})
			.catch((error) => {
				console.error('Error copying URL:', error);
				copyMessage = 'Failed to copy URL.';
				setTimeout(() => {
					copyMessage = '';
				}, 2000);
			});
	}

	function generateNewRoom() {
		room_id = Math.random().toString(36).substr(2, 9);
		window.location.hash = room_id;
	}

	function startLocalVideo() {
		navigator.mediaDevices
			.getUserMedia({
				audio: true,
				video: true
			})
			.then((stream) => {
				localVideo.srcObject = stream;
				localVideo.play();
			})
			.catch((error) => {
				message = 'Error accessing media devices.';
			});
	}

	onDestroy(() => {
		leaveRoom();
	});
</script>

<div class="min-h-screen bg-gray-900 p-4 text-gray-200">
	<div class="mx-auto w-full max-w-7xl space-y-6">
		<div class="flex flex-col justify-between gap-4 md:flex-row md:items-start">
			<!-- join -->
			<div class="w-full max-w-md rounded-lg bg-gray-800 p-6 shadow-lg">
				<div class="flex items-center justify-between gap-4">
					<input
						type="text"
						placeholder="Enter your name"
						bind:value={client_id}
						class="w-full rounded-md bg-gray-700 p-2 text-white focus:ring-2 focus:ring-sky-500 focus:outline-none"
						required
						on:keydown={(e) => e.key === 'Enter' && joinRoom()}
					/>
					<button
						on:click={joinRoom}
						class="w-40 cursor-pointer rounded-md bg-sky-700 py-2 transition hover:bg-sky-600"
					>
						Join Room
					</button>
				</div>
				{#if error}
					<p transition:slide class="mt-2 text-red-400">{error}</p>
				{/if}
			</div>

			<!-- share -->
			<div class="rounded-lg bg-gray-800 p-4 px-6 shadow-lg">
				<div class="flex items-center justify-center gap-4">
					<p class="text-center text-gray-300">
						Room ID:
						<span class="font-semibold text-sky-500">
							{room_id}
						</span>
					</p>
					<button
						on:click={copyToClipboard}
						class="cursor-pointer rounded-md bg-green-800 px-2 py-1 transition ease-in-out hover:bg-green-700"
					>
						Copy Link
					</button>
					<button
						on:click={generateNewRoom}
						class="cursor-pointer rounded-md bg-violet-700 px-2 py-1 transition ease-in-out hover:bg-violet-600"
					>
						New Room
					</button>
				</div>
				{#if copyMessage}
					<p transition:slide class="mt-1 text-center text-sm text-green-400">{copyMessage}</p>
				{/if}
			</div>
		</div>

		<div class="w-full rounded-lg bg-gray-800 p-4 text-center font-medium text-gray-300 shadow-lg">
			📢 → {message}
		</div>

		<!-- Video Container -->
		<div class="flex flex-col items-center gap-4 md:flex-row">
			<div class="relative h-64 w-full md:h-96 md:w-1/2">
				<video
					bind:this={localVideo}
					muted
					class="h-full w-full rounded-lg bg-gray-800 object-cover shadow-lg"
				>
				</video>
				<div class="bg-opacity-50 absolute bottom-2 left-2 rounded bg-black px-2 py-1 text-white">
					Me
				</div>
			</div>
			<div class="relative h-64 w-full md:h-96 md:w-1/2">
				<video
					bind:this={remoteVideo}
					class="h-full w-full rounded-lg bg-gray-800 object-cover shadow-lg"
				>
					<track kind="captions" />
				</video>
			</div>
		</div>
	</div>
</div>
