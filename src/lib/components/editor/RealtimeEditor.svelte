<script lang="ts" module>
	import * as Y from 'yjs';
	import * as awarenessProtocol from 'y-protocols/awareness';

	export type YjsInfo = {
		text: Y.Text;
		awareness: awarenessProtocol.Awareness;
	};

	/**
	 * Props that both the Monaco editor and the Codemirror editor should accept.
	 */
	export type EditorProps = {
		theme?: 'dark' | 'light';
		language?: 'cpp' | 'java' | 'py' | 'plaintext';
		inlayHints?: 'on' | 'off';
		tabSize?: number;
		// Optional compiler options for the LSP
		compilerOptions?: string;
		readOnly?: boolean;

		// If undefined, that means the editor mode is still being loaded from Firebase.
		// Any editor mode is acceptable. This is relevant for the monaco editor,
		// since it cannot unload the vim extension without reloading the page.
		editorMode?: 'normal' | 'vim';

		/**
		 * If provided, connect this editor to YJS.
		 */
		yjsInfo?: YjsInfo;
	};

	const createWebsocketInterceptorClass = (
		onMessageSyncHandler: () => void,
		onSaveHandler: () => void
	) => {
		return class WebsocketInterceptor extends WebSocket {
			send(data: string | ArrayBuffer | Blob | ArrayBufferView) {
				const messageSync = 0; // defined in y-websocket
				if (data instanceof Uint8Array && data.length > 0 && data[0] === messageSync) {
					onMessageSyncHandler();
				}
				super.send(data);
			}

			set onmessage(f: any) {
				const messageSaved = 100; // defined in ide-yjs
				super.onmessage = (m) => {
					const decoder = new Uint8Array(m.data);
					if (decoder.length > 0 && decoder[0] === messageSaved) {
						onSaveHandler();
					} else {
						f(m);
					}
				};
			}
		};
	};
</script>

<script lang="ts">
	import { WebsocketProvider } from 'y-websocket';
	import { PUBLIC_YJS_SERVER } from '$env/static/public';
	import colorFromUserId, { bgColorFromUserId } from './colorFromUserId';
	import ConnectionStatusIndicator from './ConnectionStatusIndicator.svelte';
	import type { Component } from 'svelte';

	let {
		documentId,
		userId,
		Editor,
		class: className = '',
		...props
	}: {
		documentId: string;
		/**
		 * Firebase user ID
		 */
		userId: string;

		/**
		 * The editor component to use
		 */
		Editor: Component;

		class?: string;
	} & Omit<EditorProps, 'yjsInfo'> = $props();

	let yjsInfo: YjsInfo | undefined = $state(undefined);
	let connectionStatus: 'connecting' | 'saved' | 'saving' = $state('connecting');

	$effect(() => {
		const ydocument = new Y.Doc();

		let lastUpdate = 0;
		const CustomWebsocketInterceptor = createWebsocketInterceptorClass(
			() => {
				if (!provider._synced) {
					return;
				}

				// As a sketchy hack, we'll assume every sync message sent
				// corresponds to an edit the user made.
				lastUpdate = Date.now();
				connectionStatus = 'saving';
			},
			() => {
				if (Date.now() - lastUpdate > 1000) {
					connectionStatus = 'saved';
				}
			}
		);

		const provider = new WebsocketProvider(PUBLIC_YJS_SERVER, documentId, ydocument, {
			WebSocketPolyfill: CustomWebsocketInterceptor
		});

		// Used to compute the cursor color
		provider.awareness.setLocalStateField('firebaseUserID', userId);

		// Bind Yjs to the editor model
		const monacoText = ydocument.getText('monaco');
		yjsInfo = {
			text: monacoText,
			awareness: provider.awareness
		};

		// add custom color for every selector
		provider.awareness.on(
			'change',
			({ added }: { added: Array<number>; updated: Array<number>; removed: Array<number> }) => {
				// TODO: remove styles when someone leaves (ie. removed.length > 0)
				if (added.length === 0) return;
				type UserAwarenessData = Map<
					number,
					{
						firebaseUserID?: string;
						selection: any;
					}
				>;
				let awarenessState = provider.awareness.getStates() as UserAwarenessData;
				for (let addedUserID of added) {
					let firebaseUserID =
						awarenessState.get(addedUserID)?.firebaseUserID ?? '-NPeGgrWL0zpVHHZ2aECh';
					const styleToAdd = `.yRemoteSelection-${addedUserID}, .yRemoteSelectionHead-${addedUserID} {
	            --yjs-selection-color-bg: ${bgColorFromUserId(firebaseUserID)};
	            --yjs-selection-color: ${colorFromUserId(firebaseUserID)};
	          }`;
					document.body.insertAdjacentHTML('beforeend', `<style>${styleToAdd}</style>`);
				}
			}
		);

		provider.on('status', ({ status }: { status: 'disconnected' | 'connecting' | 'connected' }) => {
			if (status === 'disconnected' || status === 'connecting') {
				connectionStatus = 'connecting';
			} else {
				connectionStatus = 'saved';
			}
		});

		return () => {
			connectionStatus = 'connecting';
			yjsInfo = undefined;
			ydocument.destroy();
			provider.destroy();
		};
	});

	let editor: any | undefined = $state(undefined);
	export const getValue = () => {
		return editor?.getValue();
	};

	let isSynced = $derived(connectionStatus !== 'connecting');
</script>

<div class={`tw-forms-disable tw-forms-disable-all-descendants relative h-full ${className}`}>
	<ConnectionStatusIndicator {connectionStatus} />
	<Editor bind:this={editor} {yjsInfo} {...props} readOnly={isSynced ? props.readOnly : true}
	></Editor>
</div>
