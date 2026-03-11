<script lang="ts">
    import { onMount, onDestroy } from "svelte";

    interface Props {
        apiBaseUrl?: string;
    }

    let { apiBaseUrl = "http://localhost:9957/api/clipboard" }: Props = $props();

    interface ClipboardItem {
        id: string;
        seed: string;
        type: "text" | "image" | "file";
        content: string;
        filename?: string;
        createdAt: string;
    }

    let seed = $state("");

    // 获取实际使用的种子（空值时使用 public）
    function getEffectiveSeed(): string {
        return seed.trim() || "public";
    }

    let textContent = $state("");
    let items: ClipboardItem[] = $state([]);
    let loading = $state(false);
    let uploading = $state(false);
    let fileInput: HTMLInputElement;
    let pollInterval: number;
    let expandedItems = $state<Set<string>>(new Set());

    const MAX_PREVIEW_LINES = 5;
    const MAX_PREVIEW_CHARS = 300;

    // 解析 base URL，提取域名和 API 路径前缀
    const API_BASE = apiBaseUrl.replace(/\/api\/clipboard$/, "");
    const API_PATH = "/api/clipboard";

    // 处理相对路径，转换为完整 URL
    function resolveUrl(path: string): string {
        if (!path) return "";
        if (path.startsWith("http://") || path.startsWith("https://")) {
            return path;
        }
        return API_BASE + path;
    }

    // 检查文本是否需要折叠
    function shouldTruncate(content: string): boolean {
        return content.length > MAX_PREVIEW_CHARS || content.split("\n").length > MAX_PREVIEW_LINES;
    }

    // 截断文本
    function truncateText(content: string): string {
        const lines = content.split("\n");
        if (lines.length > MAX_PREVIEW_LINES) {
            return lines.slice(0, MAX_PREVIEW_LINES).join("\n") + "...";
        }
        if (content.length > MAX_PREVIEW_CHARS) {
            return content.slice(0, MAX_PREVIEW_CHARS) + "...";
        }
        return content;
    }

    // 切换展开/折叠
    function toggleExpand(id: string) {
        const newSet = new Set(expandedItems);
        if (newSet.has(id)) {
            newSet.delete(id);
        } else {
            newSet.add(id);
        }
        expandedItems = newSet;
    }

    // 判断项目是否展开
    function isExpanded(id: string): boolean {
        return expandedItems.has(id);
    }

    async function fetchItems() {
        const effectiveSeed = getEffectiveSeed();
        loading = true;
        try {
            const res = await fetch(`${API_BASE}${API_PATH}/items?seed=${encodeURIComponent(effectiveSeed)}`);
            const data = await res.json();
            if (data.success) {
                items = data.data || [];
            }
        } catch (e) {
            console.error("Failed to fetch items:", e);
        } finally {
            loading = false;
        }
    }

    async function postText() {
        if (!textContent.trim()) return;
        uploading = true;
        try {
            const res = await fetch(`${API_BASE}${API_PATH}/items`, {
                method: "POST",
                headers: { "Content-Type": "application/json" },
                body: JSON.stringify({
                    seed: getEffectiveSeed(),
                    type: "text",
                    content: textContent,
                }),
            });
            const data = await res.json();
            if (data.success) {
                textContent = "";
                fetchItems();
            }
        } catch (e) {
            console.error("Failed to post text:", e);
        } finally {
            uploading = false;
        }
    }

    async function postFile(file: File) {
        uploading = true;
        try {
            const formData = new FormData();
            formData.append("seed", getEffectiveSeed());
            formData.append("type", file.type.startsWith("image/") ? "image" : "file");
            formData.append("file", file);

            const res = await fetch(`${API_BASE}${API_PATH}/items`, {
                method: "POST",
                body: formData,
            });
            const data = await res.json();
            if (data.success) {
                fetchItems();
            }
        } catch (e) {
            console.error("Failed to upload file:", e);
        } finally {
            uploading = false;
        }
    }

    async function deleteItem(id: string) {
        try {
            const res = await fetch(`${API_BASE}${API_PATH}/items/${id}`, {
                method: "DELETE",
            });
            const data = await res.json();
            if (data.success) {
                fetchItems();
            }
        } catch (e) {
            console.error("Failed to delete item:", e);
        }
    }

    // 复制文本到剪贴板
    async function copyText(text: string) {
        try {
            await navigator.clipboard.writeText(text);
        } catch (e) {
            console.error("Failed to copy:", e);
        }
    }

    // 复制图片到剪贴板
    async function copyImage(url: string) {
        try {
            const response = await fetch(url);
            const blob = await response.blob();
            await navigator.clipboard.write([
                new ClipboardItem({
                    [blob.type]: blob
                })
            ]);
        } catch (e) {
            console.error("Failed to copy image:", e);
        }
    }

    // 下载文件（强制下载，不在当前页面打开）
    async function downloadFile(url: string, filename?: string) {
        try {
            const response = await fetch(url);
            const blob = await response.blob();
            const blobUrl = URL.createObjectURL(blob);
            const link = document.createElement("a");
            link.href = blobUrl;
            link.download = filename || "";
            link.click();
            URL.revokeObjectURL(blobUrl);
        } catch (e) {
            console.error("Failed to download:", e);
            // 降级：直接打开链接
            window.open(url, "_blank");
        }
    }

    function handleFileSelect(e: Event) {
        const target = e.target as HTMLInputElement;
        const file = target.files?.[0];
        if (file) {
            postFile(file);
            target.value = "";
        }
    }

    function handlePaste(e: ClipboardEvent) {
        const items = e.clipboardData?.items;
        if (!items) return;

        for (const item of items) {
            if (item.type.startsWith("image/")) {
                const file = item.getAsFile();
                if (file) {
                    postFile(file);
                }
                e.preventDefault();
                break;
            }
        }
    }

    function formatDate(dateStr: string) {
        const date = new Date(dateStr);
        return date.toLocaleString("zh-CN", {
            month: "2-digit",
            day: "2-digit",
            hour: "2-digit",
            minute: "2-digit",
        });
    }

    function handleSeedChange() {
        fetchItems();
    }

    onMount(() => {
        fetchItems();
        pollInterval = setInterval(fetchItems, 3000);
        document.addEventListener("paste", handlePaste);
    });

    onDestroy(() => {
        if (pollInterval) clearInterval(pollInterval);
        document.removeEventListener("paste", handlePaste);
    });
</script>

<div class="card-base px-6 py-6">
    <!-- Seed Input -->
    <div class="flex flex-col sm:flex-row gap-3 mb-6">
        <input
            type="text"
            bind:value={seed}
            oninput={handleSeedChange}
            placeholder="输入种子，会有不同内容哦"
            class="flex-1 px-4 py-2 rounded-lg bg-[var(--btn-plain-bg-hover)] border border-[var(--btn-plain-bg-hover)] focus:border-[var(--primary)] focus:outline-none transition placeholder:text-30 text-75"
        />
        <button
            onclick={postText}
            disabled={uploading || !textContent.trim()}
            class="px-6 py-2 rounded-lg bg-[var(--primary)] text-black dark:text-white font-medium hover:opacity-90 disabled:opacity-50 disabled:cursor-not-allowed transition"
        >
            发布文本
        </button>
    </div>

    <!-- Text Input -->
    <div class="mb-6">
        <textarea
            bind:value={textContent}
            placeholder="在这里输入文本内容..."
            rows="3"
            class="w-full px-4 py-3 rounded-lg bg-[var(--btn-plain-bg-hover)] border border-[var(--btn-plain-bg-hover)] focus:border-[var(--primary)] focus:outline-none transition resize-none placeholder:text-30 text-75"
        ></textarea>
    </div>

    <!-- Upload Buttons -->
    <div class="flex gap-3 mb-6">
        <button
            onclick={() => fileInput.click()}
            disabled={uploading}
            class="px-4 py-2 rounded-lg border border-[var(--btn-plain-bg-hover)] text-75 hover:border-[var(--primary)] hover:text-[var(--primary)] disabled:opacity-50 disabled:cursor-not-allowed transition flex items-center gap-2"
        >
            <svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 16l4.586-4.586a2 2 0 012.828 0L16 16m-2-2l1.586-1.586a2 2 0 012.828 0L20 14m-6-6h.01M6 20h12a2 2 0 002-2V6a2 2 0 00-2-2H6a2 2 0 00-2 2v12a2 2 0 002 2z" />
            </svg>
            上传图片/文件
        </button>
        <input
            bind:this={fileInput}
            type="file"
            accept="image/*,application/pdf,.zip,.doc,.docx,.txt"
            onchange={handleFileSelect}
            class="hidden"
        />
        <span class="text-sm text-30 self-center">
            也可直接 Ctrl+V 粘贴图片
        </span>
    </div>

    <!-- Divider -->
    <div class="border-t border-[var(--btn-plain-bg-hover)] my-6"></div>

    <!-- Items List -->
    {#if loading && items.length === 0}
        <div class="text-center py-8 text-30">加载中...</div>
    {:else if items.length === 0}
        <div class="text-center py-8 text-30">
            暂无内容
            {#if !seed.trim()}
                ，请输入一个种子名称
            {:else if seed.trim() !== "public"}
                ，试试种子 "public"
            {/if}
        </div>
    {:else}
        <div class="grid gap-4">
            {#each items as item (item.id)}
                <div class="p-4 rounded-lg bg-[var(--btn-plain-bg-hover)] border border-[var(--btn-plain-bg-hover)] hover:border-[var(--primary)] transition group relative">
                    <div class="flex justify-between items-start mb-2">
                        <div class="flex items-center gap-2">
                            <span class="text-xs px-2 py-0.5 rounded bg-[var(--primary)] text-white">
                                {item.type === "text" ? "文本" : item.type === "image" ? "图片" : "文件"}
                            </span>
                            {#if item.filename}
                                <span class="text-sm text-50">{item.filename}</span>
                            {/if}
                        </div>
                        <!-- 操作按钮区域 -->
                        <div class="flex gap-2 opacity-0 group-hover:opacity-100 transition duration-200 ease-in-out">
                            {#if item.type === "text"}
                                <button
                                    onclick={() => copyText(item.content)}
                                    class="p-1.5 rounded text-30 hover:text-[var(--primary)] hover:bg-[var(--btn-plain-bg)]"
                                    title="复制"
                                >
                                    <svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M8 16H6a2 2 0 01-2-2V6a2 2 0 012-2h8a2 2 0 012 2v2m-6 12h8a2 2 0 002-2v-8a2 2 0 00-2-2h-8a2 2 0 00-2 2v8a2 2 0 002 2z" />
                                    </svg>
                                </button>
                            {:else if item.type === "image"}
                                <button
                                    onclick={() => copyImage(resolveUrl(item.content))}
                                    class="p-1.5 rounded text-30 hover:text-[var(--primary)] hover:bg-[var(--btn-plain-bg)]"
                                    title="复制图片"
                                >
                                    <svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M8 16H6a2 2 0 01-2-2V6a2 2 0 012-2h8a2 2 0 012 2v2m-6 12h8a2 2 0 002-2v-8a2 2 0 00-2-2h-8a2 2 0 00-2 2v8a2 2 0 002 2z" />
                                    </svg>
                                </button>
                                <button
                                    onclick={() => downloadFile(resolveUrl(item.content), item.filename)}
                                    class="p-1.5 rounded text-30 hover:text-[var(--primary)] hover:bg-[var(--btn-plain-bg)]"
                                    title="下载"
                                >
                                    <svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 16v1a3 3 0 003 3h10a3 3 0 003-3v-1m-4-4l-4 4m0 0l-4-4m4 4V4" />
                                    </svg>
                                </button>
                            {:else}
                                <button
                                    onclick={() => downloadFile(resolveUrl(item.content), item.filename)}
                                    class="p-1.5 rounded text-30 hover:text-[var(--primary)] hover:bg-[var(--btn-plain-bg)]"
                                    title="下载"
                                >
                                    <svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 16v1a3 3 0 003 3h10a3 3 0 003-3v-1m-4-4l-4 4m0 0l-4-4m4 4V4" />
                                    </svg>
                                </button>
                            {/if}
                            <button
                                onclick={() => deleteItem(item.id)}
                                class="p-1.5 rounded text-30 hover:text-red-500 hover:bg-[var(--btn-plain-bg)]"
                                title="删除"
                            >
                                <svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                                    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 7l-.867 12.142A2 2 0 0116.138 21H7.862a2 2 0 01-1.995-1.858L5 7m5 4v6m4-6v6m1-10V4a1 1 0 00-1-1h-4a1 1 0 00-1 1v3M4 7h16" />
                                </svg>
                            </button>
                        </div>
                    </div>

                    {#if item.type === "text"}
                        {@const needsTruncate = shouldTruncate(item.content)}
                        {@const isItemExpanded = isExpanded(item.id)}
                        <p class="text-75 whitespace-pre-wrap break-words">
                            {#if needsTruncate && !isItemExpanded}
                                {truncateText(item.content)}
                            {:else}
                                {item.content}
                            {/if}
                        </p>
                        {#if needsTruncate}
                            <button
                                onclick={() => toggleExpand(item.id)}
                                class="text-sm text-[var(--primary)] hover:underline mt-1"
                            >
                                {isItemExpanded ? "收起" : "展开全文"}
                            </button>
                        {/if}
                    {:else if item.type === "image"}
                        <div class="relative inline-block">
                            <img
                                src={resolveUrl(item.content)}
                                alt={item.filename || "图片"}
                                class="max-w-full max-h-64 rounded border border-[var(--btn-plain-bg-hover)]"
                            />
                        </div>
                    {:else}
                        <div class="flex items-center gap-2">
                            <svg class="w-5 h-5 text-30" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M7 21h10a2 2 0 002-2V9.414a1 1 0 00-.293-.707l-5.414-5.414A1 1 0 0012.586 3H7a2 2 0 00-2 2v14a2 2 0 002 2z" />
                            </svg>
                            <span class="text-50">{item.filename}</span>
                        </div>
                    {/if}

                    <div class="text-xs text-30 mt-2">
                        {formatDate(item.createdAt)}
                    </div>
                </div>
            {/each}
        </div>
    {/if}
</div>
