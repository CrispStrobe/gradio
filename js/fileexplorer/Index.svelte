<svelte:options accessors={true} />

<script lang="ts">
	import type { FileExplorerProps, FileExplorerEvents } from "./types";
	import { Gradio } from "@gradio/utils";
	import { File } from "@gradio/icons";

	import { Block, BlockLabel, IconButtonWrapper } from "@gradio/atoms";
	import DirectoryExplorer from "./shared/DirectoryExplorer.svelte";

	import { StatusTracker } from "@gradio/statustracker";

	import { _ } from "svelte-i18n";

	const props = $props();
	const gradio = new Gradio<FileExplorerEvents, FileExplorerProps>(props);

	let old_value = $state(gradio.props.value);
	let old_root = $state(gradio.props.root_dir);

	// 🛠️ FORCE RE-RENDER LOGIC
	// This creates a unique key based on the directory and patterns.
	// When these change, the {#key} block in the HTML will destroy 
	// and recreate the DirectoryExplorer from scratch.
	let rerender_key = $derived([
		gradio.props.root_dir,
		gradio.props.glob,
		gradio.props.ignore_glob
	]);

	$effect(() => {
		// 1. Handle Selection Changes
		if (old_value !== gradio.props.value) {
			old_value = gradio.props.value;
			gradio.dispatch("change");
		}

		// 2. Handle Root Directory Changes (Login/Logout Fix)
		// If the server tells us the root changed, we must clear the selection
		// to force the UI to fetch the new file list for the new user.
		if (old_root !== gradio.props.root_dir) {
			old_root = gradio.props.root_dir;
			gradio.props.value = []; // Clear selection for new root
			gradio.dispatch("change");
		}
	});
</script>

<Block
	visible={gradio.shared.visible}
	variant={gradio.props.value === null ? "dashed" : "solid"}
	border_mode={"base"}
	padding={false}
	elem_id={gradio.shared.elem_id}
	elem_classes={gradio.shared.elem_classes}
	container={gradio.shared.container}
	scale={gradio.shared.scale}
	min_width={gradio.shared.min_width}
	allow_overflow={true}
	overflow_behavior="auto"
	height={gradio.props.height}
	max_height={gradio.props.max_height}
	min_height={gradio.props.min_height}
>
	<StatusTracker
		{...gradio.shared.loading_status}
		autoscroll={gradio.shared.autoscroll}
		i18n={gradio.i18n}
		on_clear_status={() =>
			gradio.dispatch("clear_status", gradio.shared.loading_status)}
	/>
	{#if gradio.shared.show_label && gradio.props.buttons && gradio.props.buttons.length > 0}
		<IconButtonWrapper
			buttons={gradio.props.buttons}
			on_custom_button_click={(id) => {
				gradio.dispatch("custom_button_click", { id });
			}}
		/>
	{/if}
	<BlockLabel
		show_label={gradio.shared.show_label}
		Icon={File}
		label={gradio.shared.label || "FileExplorer"}
		float={false}
	/>
	{#key rerender_key}
		<DirectoryExplorer
			bind:value={gradio.props.value}
			file_count={gradio.props.file_count}
			interactive={gradio.shared.interactive}
			selectable={gradio.props._selectable}
			ls_fn={gradio.shared.server.ls}
			on:input={() => gradio.dispatch("input")}
			on:select={(e) => gradio.dispatch("select", e.detail)}
		/>
	{/key}
</Block>
