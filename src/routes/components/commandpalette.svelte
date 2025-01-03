<script lang="ts">
    import { onMount } from "svelte";
    import { invoke } from "@tauri-apps/api/core";
    import { getContext } from "svelte";

    let selectedindex: number = $state(-1);
    let searchText: string = $state("");

    const editor: any = getContext('editor');

    interface Command {
        name: string;
        shortcut: string;
        action: () => void;
    }

    let commands: Command[] = [
      {
        name: 'Delete Tab',
        shortcut: 'Ctrl + D',
        action: () => {
            editor.deleteDocument();
            editor.toggleCommandPalette();
        }
      },
      {
        name: 'New Tab',
        shortcut: 'Ctrl + N',
        action: () => {
            editor.newDocument();
            editor.toggleCommandPalette();
        }
      },
      {
        name: 'Next Tab',
        shortcut: 'Ctrl + Tab or Ctrl + pgDown',
        action: () => {
            editor.cycleTabs();
            editor.toggleCommandPalette();
        }
      },
      {
        name: 'Go to First Tab',
        shortcut: 'Ctrl + 1',
        action: () => {
            editor.gotoTab1();
            editor.toggleCommandPalette();
        }
      },
      {
        name: 'Go to Last Tab',
        shortcut: 'Ctrl + 9',
        action: () => {
            editor.gotoLastTab();
            editor.toggleCommandPalette();
        }
      },
      {
        name: 'Toggle ToolBar',
        shortcut: 'Ctrl + T',
        action: () => {
            editor.toggleToolbar();
            editor.toggleCommandPalette();
        }
      }
    ];

    function handleKeydown(event: KeyboardEvent) {
        if (!editor.return_isCommandPalettevisible()) return;

        switch (event.key) {
            case 'ArrowDown':
                event.preventDefault();
                if (selectedindex === -1) {
                    selectedindex = 0;
                } else {
                    selectedindex = (selectedindex + 1) % commands.length;
                }
                break;
            case 'ArrowUp':
                event.preventDefault();
                if (selectedindex === -1) {
                    selectedindex = commands.length - 1;
                } else {
                    selectedindex = (selectedindex - 1 + commands.length) % commands.length;
                }
                break;
            case 'Enter':
                event.preventDefault();
                if (selectedindex >= 0 && selectedindex < commands.length) {
                    commands[selectedindex].action();
                }
                break;
            case 'Escape':
                event.preventDefault();
                editor.toggleCommandPalette();
                break;
        }
    }

    function handleWheel(event: WheelEvent) {
        if (!editor.return_isCommandPalettevisible()) return;
        
        event.preventDefault();
        if (event.deltaY > 0) {
            // Scrolling down
            selectedindex = (selectedindex + 1) % commands.length;
        } else {
            // Scrolling up
            selectedindex = (selectedindex - 1 + commands.length) % commands.length;
        }
    }

    // Reset selected index when command palette is closed
    $effect(() => {
        if (!editor.return_isCommandPalettevisible()) {
            selectedindex = -1;
            searchText = "";
        }
    });
</script>

<svelte:window on:keydown={handleKeydown} />

<main>
    {#if editor.return_isCommandPalettevisible()}
        <!-- svelte-ignore a11y_no_static_element_interactions -->
        <!-- svelte-ignore a11y_click_events_have_key_events -->
        <div 
            class="background-blur" 
            onclick={() => editor.toggleCommandPalette()}
            >
        </div>
        <div class="commandPalette" onwheel={handleWheel}>
            <div class="search-container">
                <textarea
                    class="command-search"
                    placeholder="Select a Command"
                    bind:value={searchText}
                ></textarea>
                <button 
                    class="close-button"
                    onclick={() => editor.toggleCommandPalette()}
                >
                    ✕
                </button>
            </div>
            {#each commands as command, index}
                <button
                    type="button"
                    class="command-item"
                    class:active={selectedindex === index}
                    onclick={() => {
                        command.action();
                    }}
                    onmouseenter={() => selectedindex = index}
                >
                    <span>{command.name}</span>
                    <span class="shortcut">{command.shortcut}</span>
                </button>
            {/each}
        </div>
    {/if}
</main>
