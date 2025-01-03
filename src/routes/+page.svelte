<script lang="ts">
    import { onMount } from "svelte";
    import { invoke } from "@tauri-apps/api/core";
    import "../styles/styles.css";
    import { v4 as uuidv4 } from "uuid";
    import Quill from "quill";
    import { setContext } from 'svelte';
    import Commandpalette from "./components/commandpalette.svelte";

    interface Tab {
        id: string;
        title: string;
    }

    interface Document {
        id: string;
        title: string;
        content: string;
    }

    interface QuickFormatOption {
        icon: string;
        format: string;
        value?: any;
    }

    interface ToolbarOptions {
        [key: string]: any;
    }

    // State management
    let titleText: string = $state("");
    let textboxText: string = "";
    let recentDocuments: Document[] = [];
    let currentId: string = $state("");
    let wordCount: number = $state(0);
    let charCount: number = $state(0);
    let isToolbarVisible: boolean = $state(false);
    let isCommandPalettevisible: boolean = $state(false);
    let currentTabs: Tab[] = $state([])
    // let tabCount: number = $state(1);
    // let tabs: Tab[] = $state([]);

    //setcontext for the editor to pass to the child components:
    setContext(
        'editor', 
        {   
            addnewtab, 
            newDocument,
            switchTab, 
            gotoLastTab, 
            gotoTab1, 
            cycleTabs, 
            deleteDocument, 
            toggleToolbar, 
            return_isCommandPalettevisible: () => isCommandPalettevisible, 
            toggleCommandPalette 
    });

    // Initialize Quill
    let quill: Quill;
    const toolbarOptions: any[] = [
        ["bold", "italic", "underline", "strike"],
        ["blockquote", "code-block"],
        [{ header: 1 }, { header: 2 }],
        [{ list: "ordered" }, { list: "bullet" }],
        [{ script: "sub" }, { script: "super" }],
        [{ indent: "-1" }, { indent: "+1" }],
        [{ direction: "rtl" }],
        [{ size: ["small", false, "large", "huge"] }],
        [{ header: [1, 2, 3, 4, 5, 6, false] }],
        [{ color: [] }, { background: [] }],
        [{ font: [] }],
        [{ align: [] }],
        ["clean"],
        ["link", "image", "video"],
    ];

    let customTooltip: HTMLDivElement;
    const quickFormatOptions: QuickFormatOption[] = [
        { icon: "B", format: "bold" },
        { icon: "I", format: "italic" },
        { icon: "U", format: "underline" },
        { icon: "S", format: "strike" },
        { icon: "¶", format: "blockquote" },
        { icon: "<>", format: "code-block" },
    ];

    onMount(() => {
        // Initialize Quill editor
        quill = new Quill("#editor", {
            theme: "snow",
            placeholder: "start typing...",
            modules: {
                toolbar: toolbarOptions,
            },
            bounds: "#editor",
        });

        const editorElement = document.querySelector("#editor");
        const toolbarElement = document.querySelector(".ql-toolbar");

        if (editorElement) {
            editorElement.classList.add("quill-dark-theme");
        }

        if (toolbarElement instanceof HTMLElement) {
            toolbarElement.style.display = "none";
        }

        // Create custom tooltip element
        customTooltip = document.createElement("div");
        customTooltip.className = "ql-tooltip";
        document.body.appendChild(customTooltip);

        // Add selection change handler
        quill.on(
            "selection-change",
            (range: { index: number; length: number } | null) => {
                if (range && range.length > 0) {
                    const bounds = quill.getBounds(range.index, range.length);
                    const quillContainer = document.querySelector("#editor");

                    if (quillContainer instanceof HTMLElement) {
                        const containerBounds =
                            quillContainer.getBoundingClientRect();

                        customTooltip.style.position = "absolute";
                        customTooltip.style.left = `${containerBounds.left + bounds!.left}px`;
                        customTooltip.style.top = `${containerBounds.top + bounds!.top - 40}px`;

                        // Clear existing buttons
                        customTooltip.innerHTML = "";

                        // Add format buttons
                        quickFormatOptions.forEach((option) => {
                            const button = document.createElement("button");
                            button.textContent = option.icon;
                            button.onclick = () => {
                                if (option.value !== undefined) {
                                    quill.format(option.format, option.value);
                                } else {
                                    quill.format(
                                        option.format,
                                        !quill.getFormat()[option.format],
                                    );
                                }
                            };
                            customTooltip.appendChild(button);
                        });

                        customTooltip.style.display = "flex";
                    }
                } else {
                    customTooltip.style.display = "none";
                }
            },
        );

        quill.on("text-change", () => {
            const text = quill.getText() || "";
            wordCount = countWords(text);
            charCount = Math.max(0, text.length - 1);
        });

        const toolbar = document.querySelector(".ql-toolbar");
        if (toolbar instanceof HTMLElement) {
            toolbar.style.display = "none";
            toolbar.classList.remove("visible");
        }

        loadRecentDocuments().then(async () => {
            await updateTabs();
            if (currentTabs.length === 0) {
                await addnewtab();
            }
        });

        // Set up auto-save
        const autoSaveInterval = setInterval(autoSave, 500);

        return () => {
            // Cleanup
            customTooltip?.remove();
            clearInterval(autoSaveInterval);
        };
    });

    function countWords(text: string): number {
        return text.split(/\s+/).filter((word) => word.length > 0).length;
    }

    async function getTabs(): Promise<Tab[]> {
        return await invoke("get_tabs"); // New Rust function needed
    }

    async function updateTabs(): Promise<void> {
        currentTabs = await getTabs();
    }

    async function addnewtab(): Promise<void> {
        const newTab: Tab = await invoke("new_tab");
        currentId = newTab.id;
        titleText = newTab.title;
        quill?.setContents([]);
        await updateTabs();
        await invoke("send_current_open_tab", { id: newTab.id });
    }

    async function switchTab(tabId: string): Promise<void> {
        try {
            const docResult: Document | null = await invoke(
                "get_document_content",
                { id: tabId }
            );

            if (docResult) {
                currentId = tabId;
                titleText = docResult.title;
                quill?.setContents(JSON.parse(docResult.content));
            } else {
                currentId = tabId;
                titleText = "";
                quill?.setContents([]);
            }
            
            // Update the current open tab in the backend
            await invoke("send_current_open_tab", { id: tabId });
        } catch (error) {
            console.error("Failed to switch tab:", error);
        }
    }

    async function cycleTabs(): Promise<void> {
        try {
            const nextTabId: string = await invoke("cycle_tabs");
            const docResult: Document | null = await invoke("get_document_content", { id: nextTabId });
            
            if (docResult) {
                currentId = nextTabId;
                titleText = docResult.title;
                quill?.setContents(JSON.parse(docResult.content));
            }
        } catch (error) {
            console.error("Failed to cycle tabs:", error);
        }
    }

    async function gotoTab1(): Promise<void> {
        const tabs = await getTabs();
        if (tabs.length > 0) {
            await switchTab(tabs[0].id);
        }
    }

    async function gotoLastTab(): Promise<void> {
        const tabs = await getTabs();
        if (tabs.length > 0) {
            await switchTab(tabs[tabs.length - 1].id);
        }
    }

    async function autoSave(): Promise<void> {
        if (!titleText && !quill?.getText().trim()) return;

        try {
            await invoke("update_tab_title", {
                id: currentId,
                title: titleText,
            });
            await updateTabs();
            await invoke("save_document", {
                id: currentId,
                title: titleText,
                content: JSON.stringify(quill.getContents()),
            });
        } catch (error) {
            console.error("Auto-save failed:", error);
        }
    }

    async function loadRecentDocuments(): Promise<void> {
        try {
            const docs: Document[] = await invoke("load_recent_files");
            recentDocuments = docs;

            // Update the tabs in UI
            await updateTabs();

            if (recentDocuments.length > 0) {
                // Load the last open tab from the backend
                const openTabId: string = await invoke("get_current_open_tab");
                await switchTab(openTabId);
            } else {
                // If no documents exist, create a new tab
                await addnewtab();
            }
        } catch (error) {
            console.error("Failed to load documents:", error);
        }
    }

    function handleTitleChange(event: Event): void {
        const target = event.target as HTMLTextAreaElement;
        titleText = target.value;
    }

    function handleKeydown(event: KeyboardEvent): void {
        if (event.ctrlKey && event.key === "d") {
            event.preventDefault();
            deleteDocument();
        }
        if (event.ctrlKey && event.key === "n") {
            event.preventDefault();
            newDocument();
        }
        if (event.ctrlKey && event.key === "t") {
            event.preventDefault();
            toggleToolbar();
        }
        if (
            (event.ctrlKey && event.key === "Tab") ||
            (event.ctrlKey && event.key === "PageDown")
        ) {
            event.preventDefault();
            cycleTabs();
        }
        if (event.ctrlKey && event.key === "1") {
            event.preventDefault();
            gotoTab1();
        }
        if (event.ctrlKey && event.key === "9") {
            event.preventDefault();
            gotoLastTab();
        }
        if (event.ctrlKey && event.key === "p") {
            event.preventDefault();
            toggleCommandPalette();
        }
    }

    function toggleCommandPalette(): void {
        isCommandPalettevisible = !isCommandPalettevisible;
    }

    function toggleToolbar(): void {
        isToolbarVisible = !isToolbarVisible;
        const toolbar = document.querySelector(".ql-toolbar");
        if (toolbar instanceof HTMLElement) {
            if (isToolbarVisible) {
                toolbar.style.display = "block";
                setTimeout(() => {
                    toolbar.classList.add("visible");
                }, 10);
            } else {
                toolbar.classList.remove("visible");
                setTimeout(() => {
                    toolbar.style.display = "none";
                }, 300);
            }
        }
    }

    async function deleteDocument(): Promise<void> {
        try {
            // The Rust function returns the next document's content after deletion
            const nextDoc: Document | null = await invoke("delete_document", { id: currentId });
            await updateTabs();
            
            if (nextDoc) {
                // If we have a next document, switch to it
                currentId = nextDoc.id;
                titleText = nextDoc.title;
                quill?.setContents(JSON.parse(nextDoc.content));
            } else {
                // If no documents left, create a new one
                await addnewtab();
            }
            
            await invoke("send_current_open_tab", { id: currentId });
        } catch (error) {
            console.error("Failed to delete document:", error);
        }
    }

    async function newDocument(): Promise<void> {
        try {
            const newTab: Tab = await invoke("new_tab");
            currentId = newTab.id;
            titleText = newTab.title;
            quill?.setContents([]);
            await updateTabs();
        } catch (error) {
            console.error("Failed to create new document:", error);
        }
    }

    // Update word and character counts when text changes
    $effect(() => {
        if (quill) {
            const text = quill.getText() || "";
            wordCount = countWords(text);
            charCount = Math.max(0, text.length - 1);
        }
    });
</script>

<svelte:window on:keydown={handleKeydown} />

<main>
    <div class="container">
        <div class="title-container title-textarea">
            <textarea
                class="rounded-container"
                placeholder="Enter Title here..."
                value={titleText}
                oninput={handleTitleChange}
            ></textarea>
        </div>

        <div id="editor" class="quillbox-container">
            <div
                class="ql-toolbar ql-snow"
                class:visible={isToolbarVisible}
            ></div>
            <div class="ql-container ql-snow"></div>
        </div>
    </div>

    <div class="word-char-counter">
        <div>{wordCount} Words</div>
        <div>{charCount} Characters</div>
    </div>
    
    <div class="tab-counter" role="tablist" aria-label="Document tabs">
        {#each currentTabs as tab}
            <button
                type="button"
                class="tab-square"
                class:active={currentId === tab.id}
                role="tab"
                aria-selected={currentId === tab.id}
                aria-controls="editor"
                onclick={() => switchTab(tab.id)}
            >
                {tab.title.length > 10 ? tab.title.slice(0, 15) + '...' : tab.title || 'Untitled'}
            </button>
        {/each}
    </div>
    <Commandpalette />
</main>
