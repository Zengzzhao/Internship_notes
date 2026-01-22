<template>
    <div class="content-editor-wrapper" ref="contentWrapperRef" v-bind="$attrs" :data-editor-id="editorId" :style="{ border: props.showIcon ? 'none' : '1px solid #ebedf0' }">
        <!-- PromptParameterEditor 现在通过 CSS 固定在右上角 -->
        <PromptParameterEditor
            v-if="props.showParameterEditor"
            class="prompt-editor-single-var"
            v-show="showPromptParameterEditor"
            ref="promptEditorRef"
            :data="editingParams"
            :showMessage="props.showParamError"
            @change="handlePromptEditorChange"
            @generate-variables-click="handleGenerate"
        ></PromptParameterEditor>

        <!-- <div class="editor-toolbar">
            <button ref="fillAllVarsBtn" @click.stop="openEditorForAllVariables">填写全部变量</button>
            <div class="undo-redo-buttons">
                <button class="undo-btn" :disabled="!canUndo" @click="undo" title="撤销 (Ctrl+Z)">↶ 撤销</button>
                <button class="redo-btn" :disabled="!canRedo" @click="redo" title="重做 (Ctrl+Y)">↷ 重做</button>
            </div>
        </div> -->
        <div
            ref="editorDiv"
            class="editor-area editor"
            :class="[
                isNoVar && props.varAlertInfo ? 'editor-area-without-var' : '',
                props.disabled ? 'editor-disabled' : '',
            ]"
            :contenteditable="!props.disabled"
            @input="handleInput"
            @click="handleEditorClick"
            @paste="handlePaste"
            @compositionstart="handleCompositionStart"
            @compositionend="handleCompositionEnd"
            @keydown="handleKeyDown"
        ></div>
        <div
            ref="editorDiv2"
            class="editor-area editor"
            :class="isNoVar && props.varAlertInfo ? 'editor-area-without-var' : ''"
            contenteditable="false"
            style="position: absolute; top: 0; left: 0; width: 100%; height: 100%; z-index: -1; overflow-y: auto;"
        ></div>
        <div class="alert-info-wrapper">
            <el-alert
                v-if="isNoVar && props.varAlertInfo"
                class="alert-info"
                :title="noVarAlertInfo"
                type="warning"
                :closable="false"
                show-icon
            />
        </div>
        <button
            ref="optimizeBtn"
            v-if="false"
            class="optimize-btn"
            style="display: none !important"
            @mousedown.prevent="onOptimizeClick"
        >
            优化
        </button>


    </div>

    <div ref="editorHidden" style="display: none"></div>
</template>

<script setup lang="ts">
defineOptions({
    inheritAttrs: false,
});
interface WorkbenchData {
    [key: string]: string;
}
import { ref, onMounted, onUnmounted, nextTick, computed, watch } from 'vue';
import PromptParameterEditor from '@/components/promptVarEditor.vue'; // 确保路径正确
import { escape } from 'lodash-es';

export interface Variable {
    key: string;
    value: string;
    type: string;
}

export interface ChangeEventData {
    contentWithVar: string;
    contentRealVar: string;
    variables: Variable[];
}

// 定义 props
interface Props {
    promptTxt?: string;
    showParamError?: boolean;
    varAlertInfo?: boolean;
    showParameterEditor?: boolean;
    disabled?: boolean;
    showIcon?: boolean;
}

const props = withDefaults(defineProps<Props>(), {
    varAlertInfo: true,
    promptTxt: '',
    disabled: false,
});
// 定义 emits
const emit = defineEmits<{
    (e: 'change', data: ChangeEventData): void;
    (e: 'generate-variables-click'): void;
    (e: 'empty-braces-typed'): void;
}>();

const isNoVar = ref(false);
const editorId = ref(`prompt-editor-${Math.random().toString(36).substring(2, 15)}`);

// 保持一份 promptTxt 的内部副本
const promptTxtCopy = ref(props.promptTxt);
// 记录上一次的纯文本内容（包含 {{var}} 占位符）
const lastContentWithVar = ref('');

// 用于防止循环更新的标志
const isInternalUpdate = ref(false);

// 监听外部 promptTxt 变化，并使用 setContent 更新编辑器
watch(
    () => props.promptTxt,
    newValue => {
        // 如果是内部更新触发的，忽略
        if (isInternalUpdate.value) {
            return;
        }

        if (newValue !== promptTxtCopy.value && editorDiv.value) {
            // 检查新值是否与当前编辑器内容相同，避免不必要的重新渲染
            const currentContent = getContent(false);
            if (newValue === currentContent) {
                promptTxtCopy.value = newValue ?? '';
                return;
            }

            promptTxtCopy.value = newValue ?? '';
            const tempDiv = document.createElement('div');
            tempDiv.textContent = promptTxtCopy.value; // 防注入
            editorDiv.value.innerHTML = tempDiv.innerHTML;
            nextTick(() => {
                processContent();
                if (!historyStack.value.length) saveToHistory();
            });
        }
    },
    { immediate: true }, // 立即执行以便初始化
);

// PromptParameterEditor 组件的引用
const promptEditorRef = ref<InstanceType<typeof PromptParameterEditor> | null>(null);
// 编辑器 DOM 引用
const editorDiv = ref<HTMLDivElement | null>(null);
// 优化按钮 DOM 引用
const optimizeBtn = ref<HTMLButtonElement | null>(null);
// 换行提示元素引用与计时器
const contentWrapperRef = ref<HTMLDivElement | null>(null);
// "填写全部变量"按钮 DOM 引用 (虽然现在用不上它的位置，但 ref 可能被其他地方使用)
const fillAllVarsBtn = ref<HTMLButtonElement | null>(null);
const editorDiv2 = ref<HTMLDivElement | null>(null);
// 内部维护的变量列表
const variables = ref<Variable[]>([]);
// 用于处理富文本内容转换的隐藏 div
const editorHidden = ref<HTMLDivElement | null>(null);
// SVG 编辑图标 (用于变量按钮)
const editIconSVG = `<svg xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" contentScriptType="text/ecmascript" fill="none" width="16" zoomAndPan="magnify" contentStyleType="text/css" viewBox="0 0 16 16" height="16" preserveAspectRatio="xMidYMid meet" version="1.0"><path fill="#999999" d="M9.72656 1.73035C10.1151 1.32893 10.7573 1.32359 11.1523 1.71863L14.3105 4.87683C14.7056 5.27191 14.7004 5.91508 14.2988 6.30359L7.37793 12.9999H13.75C14.1642 12.9999 14.5 13.3357 14.5 13.7499C14.4999 14.164 14.1642 14.4999 13.75 14.4999H3.25C3.21144 14.4999 3.17371 14.4957 3.13672 14.4901H2.53906C1.98688 14.49 1.53906 14.0423 1.53906 13.4901V10.5966C1.53914 10.3372 1.63997 10.0877 1.82031 9.90124L9.72656 1.73035ZM3.00098 10.8993V12.9999H5.12988L12.9082 5.59558L10.4336 3.12097L3.00098 10.8993Z"/></svg>`;

// 正则表达式匹配 {{key}}
const VAR_RE = /\{\{([^{}]+?)\}\}/g;

const noVarAlertInfo = `Prompt 未设置变量，输入{{变量名}}插入变量，以便更精准调试生成效果。`;

// 递归遍历DOM节点获取文本内容，保留原始的换行符
const getTextContentFromNode = (node: Node, isRoot: boolean = false): string => {
    if (!node) return '';

    if (node.nodeType === Node.TEXT_NODE) {
        return node.textContent || '';
    }

    if (node.nodeType === Node.ELEMENT_NODE) {
        const element = node as HTMLElement;

        // 处理 <br> 标签，转换为换行符
        if (element.tagName.toLowerCase() === 'br') {
            return '\n';
        }

        // 处理块级元素，在前后添加换行符
        const blockElements = ['div', 'p', 'h1', 'h2', 'h3', 'h4', 'h5', 'h6', 'ul', 'ol', 'li', 'blockquote', 'pre'];
        const isBlockElement = blockElements.includes(element.tagName.toLowerCase());

        let result = '';

        // 遍历子节点
        for (let i = 0; i < node.childNodes.length; i++) {
            result += getTextContentFromNode(node.childNodes[i]);
        }

        // 对于块级元素，如果不是第一个元素且前面没有换行符，则添加换行符
        if (!isRoot && isBlockElement && result && !result.startsWith('\n')) {
            result = '\n' + result;
        }

        // 对于根节点，去除首尾空格，避免清空输入时浏览器默认插入的 br 标签被转义成 \n
        if (isRoot) {
            result = result.trim();
        }

        return result;
    }

    return '';
};

// --- PromptParameterEditor 相关状态 ---
const showPromptParameterEditor = ref(false); // 控制 PromptParameterEditor 的显示/隐藏
const editingParams = ref<WorkbenchData>({}); // 传递给 PromptParameterEditor 的数据
const clickedVarKeyForFocus = ref<string | null>(null); // 记录被点击的变量key，用于聚焦

// 将变量数组转换为 PromptParameterEditor 需要的 WorkbenchData 格式
const variablesToWorkbenchData = (vars: Variable[]): WorkbenchData => {
    return vars.reduce((acc, v) => {
        acc[v.key] = v.value;
        return acc;
    }, {} as WorkbenchData);
};

// 计算属性：获取用变量替换后的最终内容
const contentWithVariablesReplaced = () => {
    if (!editorHidden.value || !editorDiv.value) return '';

    // 获取包含 {{var}} 占位符的原始文本内容
    let content = editorHidden.value.innerText || '';

    // 用 variables 中的值替换对应的 {{key}}
    variables.value.forEach(variable => {
        const regex = new RegExp(`\\{\\{${variable.key.replace(/[.*+?^${}()|[\]\\]/g, '\\$&')}\\}\\}`, 'g');
        content = content.replace(regex, variable.value);
    });

    // 将剩余的所有 {{...}} 替换为空字符串
    content = content.replace(/\{\{[^{}]*\}\}/g, '');

    return content;
};

// 发出 change 事件，通知父组件内容或变量已更改
const emitChange = () => {
    if (!editorHidden.value || !editorDiv.value) return;

    // 获取包含 {{var}} 占位符的原始文本内容
    // 使用 editorHidden 来获取不含控制按钮的纯净HTML，再转为文本
    editorHidden.value.innerHTML = editorDiv.value.innerHTML;
    editorHidden.value.querySelectorAll('.target-set-var').forEach(el => el.remove()); // 移除编辑按钮span
    editorHidden.value.querySelectorAll('.target-var').forEach(el => {
        // 将变量span替换为其内部文本
        el.outerHTML = el.innerHTML;
    });

    // 直接从DOM节点遍历获取文本内容，保留原始的换行符
    const contentWithVar = getTextContentFromNode(editorHidden.value, true);

    // 获取用实际变量值替换占位符后的文本内容
    let contentRealVar = contentWithVar;
    variables.value.forEach(variable => {
        // 需要精确匹配 {{ key }}，包括可能的空格，并正确处理正则特殊字符
        const regex = new RegExp(`\\{\\{${variable.key.replace(/[.*+?^${}()|[\]\\]/g, '\\$&')}\\}\\}`, 'g');
        contentRealVar = contentRealVar.replace(regex, variable.value);
    });

    const changeData: ChangeEventData = {
        contentWithVar,
        contentRealVar,
        variables: JSON.parse(JSON.stringify(variables.value)), // 返回变量的深拷贝
    };

    isNoVar.value = changeData.variables.length === 0;

    // 设置内部更新标志，防止 watch 触发循环更新
    isInternalUpdate.value = true;
    promptTxtCopy.value = contentWithVar; // 同步内部副本

    emit('change', changeData);
    // 检测是否刚刚插入了 "{}"
    try {
        const oldStr = lastContentWithVar.value || '';
        const newStr = contentWithVar || '';
        const lenDiff = newStr.length - oldStr.length;
        if (lenDiff === 2) {
            let start = 0;
            while (start < oldStr.length && oldStr[start] === newStr[start]) start++;
            let endOld = oldStr.length - 1;
            let endNew = newStr.length - 1;
            while (endOld >= start && oldStr[endOld] === newStr[endNew]) { endOld--; endNew--; }
            if (newStr.slice(start, endNew + 1) === '{}') {
                emit('empty-braces-typed');
            }
        }
    } catch (e) {
        // 安静失败，避免影响主流程
    }
    // 更新上一次内容快照
    lastContentWithVar.value = contentWithVar;

    // 在下一个 tick 重置标志
    nextTick(() => {
        isInternalUpdate.value = false;
    });
};

// --- 控制 PromptParameterEditor 显示的统一函数 ---
const showVariablesEditor = (keyToFocus?: string) => {
    editingParams.value = variablesToWorkbenchData(variables.value);
    showPromptParameterEditor.value = true;
    clickedVarKeyForFocus.value = keyToFocus || null; // 保存需要聚焦的key

    nextTick(() => {
        // PromptParameterEditor 通过 CSS 固定位置，无需在此动态设置 top/left

        // 如果指定了 keyToFocus，则调用子组件的方法进行聚焦和滚动
        if (showPromptParameterEditor.value && promptEditorRef.value && clickedVarKeyForFocus.value) {
            promptEditorRef.value.focusAndScrollToVariable(clickedVarKeyForFocus.value);
            // clickedVarKeyForFocus.value = null; // 清除，避免影响下次打开
        }
    });
};

// "填写全部变量"按钮点击处理
const openEditorForAllVariables = (keyToFocus?: string) => {
    showVariablesEditor(keyToFocus); // 调用统一的显示函数，不指定特定key聚焦
};

// 中文输入法状态跟踪
const isComposing = ref(false);

const handleCompositionStart = () => {
    isComposing.value = true;
    // 输入法开始输入时，立即隐藏提示

};

const handleCompositionEnd = (e: CompositionEvent) => {
    isComposing.value = false;
    // 输入法完成后，需要手动触发 input 处理
    nextTick(() => {
        handleInput();
    });
};

// --- Undo/Redo 相关 ---
interface EditorState {
    html: string;
    variables: Variable[];
    cursorPosition?: {
        startOffset: number;
        endOffset: number;
        startPath: number[];
        endPath: number[];
    };
}

const historyStack = ref<EditorState[]>([]);
const historyIndex = ref(-1);
const maxHistorySize = 50;
const isUndoRedoOperation = ref(false);

const canUndo = computed(() => historyIndex.value > 0);
const canRedo = computed(() => historyIndex.value < historyStack.value.length - 1);

const saveToHistory = () => {
    if (!editorDiv.value || isUndoRedoOperation.value) return;

    const currentState: EditorState = {
        html: editorDiv.value.innerHTML,
        variables: JSON.parse(JSON.stringify(variables.value)),
        cursorPosition: getCurrentCursorPosition(),
    };

    if (historyStack.value.length > 0) {
        const lastState = historyStack.value[historyIndex.value];
        if (
            lastState &&
            lastState.html === currentState.html &&
            JSON.stringify(lastState.variables) === JSON.stringify(currentState.variables)
        ) {
            return; // 如果内容和变量都未改变，则不保存
        }
    }

    if (historyIndex.value < historyStack.value.length - 1) {
        historyStack.value = historyStack.value.slice(0, historyIndex.value + 1);
    }

    historyStack.value.push(currentState);
    historyIndex.value = historyStack.value.length - 1;

    if (historyStack.value.length > maxHistorySize) {
        historyStack.value.shift();
        historyIndex.value--;
    }
};

const getCurrentCursorPosition = (): EditorState['cursorPosition'] | undefined => {
    const selection = window.getSelection();
    if (
        !selection ||
        !selection.rangeCount ||
        !editorDiv.value ||
        !editorDiv.value.contains(selection.anchorNode) ||
        !editorDiv.value.contains(selection.focusNode)
    )
        return undefined;

    const range = selection.getRangeAt(0);

    try {
        return {
            startOffset: range.startOffset,
            endOffset: range.endOffset,
            startPath: getNodePath(range.startContainer, editorDiv.value),
            endPath: getNodePath(range.endContainer, editorDiv.value),
        };
    } catch (e) {
        console.warn('获取光标位置失败:', e);
        return undefined;
    }
};

const getNodePath = (node: Node, container: Node): number[] => {
    const path: number[] = [];
    let current = node;
    while (current && current !== container) {
        const parent = current.parentNode;
        if (!parent) break;
        const index = Array.from(parent.childNodes).indexOf(current as ChildNode);
        if (index === -1) break; // 如果找不到节点（例如节点已被移除），则停止
        path.unshift(index);
        current = parent;
    }
    return path;
};
// 将光标置于编辑器末尾
const setEditorFocus = () => {
    if (editorDiv.value) {
        editorDiv.value.focus();
        const selection = window.getSelection();
        const range = document.createRange();
        range.selectNodeContents(editorDiv.value);
        range.collapse(false); // false 表示折叠到末尾
        selection?.removeAllRanges();
        selection?.addRange(range);
    }
};

const restoreCursorPosition = (cursorPosition?: EditorState['cursorPosition']) => {
    if (!cursorPosition || !editorDiv.value) return;
    try {
        const startNode = getNodeByPath(cursorPosition.startPath, editorDiv.value);
        const endNode = getNodeByPath(cursorPosition.endPath, editorDiv.value);

        if (startNode && endNode) {
            const selection = window.getSelection();
            const range = document.createRange();

            // 确保 offset 不超过节点长度
            const startOffset = Math.min(cursorPosition.startOffset, startNode.textContent?.length ?? 0);
            const endOffset = Math.min(cursorPosition.endOffset, endNode.textContent?.length ?? 0);

            range.setStart(startNode, startOffset);
            range.setEnd(endNode, endOffset);

            selection?.removeAllRanges();
            selection?.addRange(range);
        }
    } catch (e) {
        console.warn('恢复光标位置失败:', e);
        // 作为后备，尝试将光标置于编辑器末尾
        setEditorFocus();
    }
};

const getNodeByPath = (path: number[], container: Node): Node | null => {
    let current: Node | null = container;
    for (const index of path) {
        if (current && current.childNodes && current.childNodes[index]) {
            current = current.childNodes[index];
        } else {
            return null; // 路径无效
        }
    }
    return current;
};

const undo = () => {
    if (!canUndo.value) return;
    isUndoRedoOperation.value = true;
    historyIndex.value--;
    const state = historyStack.value[historyIndex.value];
    if (state && editorDiv.value) {
        editorDiv.value.innerHTML = state.html;
        variables.value = JSON.parse(JSON.stringify(state.variables));
        nextTick(() => {
            restoreCursorPosition(state.cursorPosition);
            isUndoRedoOperation.value = false;
            emitChange();
        });
    } else {
        isUndoRedoOperation.value = false;
    }
};

const redo = () => {
    if (!canRedo.value) return;
    isUndoRedoOperation.value = true;
    historyIndex.value++;
    const state = historyStack.value[historyIndex.value];
    if (state && editorDiv.value) {
        editorDiv.value.innerHTML = state.html;
        variables.value = JSON.parse(JSON.stringify(state.variables));
        nextTick(() => {
            restoreCursorPosition(state.cursorPosition);
            isUndoRedoOperation.value = false;
            emitChange();
        });
    } else {
        isUndoRedoOperation.value = false;
    }
};

const handleKeyDown = (event: KeyboardEvent) => {
    // 处理删除键（Backspace 和 Delete），整体删除变量
    if (event.key === 'Backspace' || event.key === 'Delete') {
        const selection = window.getSelection();
        if (!selection || !selection.rangeCount || !editorDiv.value) return;

        const range = selection.getRangeAt(0);

        // 如果有选中内容，让默认行为处理
        if (!range.collapsed) {
            return;
        }

        const isBackspace = event.key === 'Backspace';
        let nodeToCheck: Node | null = null;

        if (isBackspace) {
            // Backspace: 检查光标前面的节点
            if (range.startOffset === 0 && range.startContainer.previousSibling) {
                nodeToCheck = range.startContainer.previousSibling;
            } else if (range.startContainer.nodeType === Node.TEXT_NODE && range.startOffset > 0) {
                // 光标在文本节点中间，不处理
                return;
            } else if (range.startContainer.nodeType === Node.ELEMENT_NODE) {
                nodeToCheck = range.startContainer.childNodes[range.startOffset - 1] || range.startContainer.previousSibling;
            }
        } else {
            // Delete: 检查光标后面的节点
            if (range.startContainer.nodeType === Node.TEXT_NODE) {
                const textNode = range.startContainer as Text;
                if (range.startOffset < (textNode.textContent?.length || 0)) {
                    // 光标在文本节点中间，不处理
                    return;
                }
                nodeToCheck = range.startContainer.nextSibling;
            } else if (range.startContainer.nodeType === Node.ELEMENT_NODE) {
                nodeToCheck = range.startContainer.childNodes[range.startOffset] || range.startContainer.nextSibling;
            }
        }

        // 向上查找，看是否是变量相关的元素
        let targetVarElement: HTMLElement | null = null;
        let targetSetVarElement: HTMLElement | null = null;
        let currentNode: Node | null = nodeToCheck;

        while (currentNode && currentNode !== editorDiv.value) {
            if (currentNode.nodeType === Node.ELEMENT_NODE) {
                const element = currentNode as HTMLElement;
                if (element.classList.contains('target-var')) {
                    targetVarElement = element;
                    break;
                } else if (element.classList.contains('target-set-var')) {
                    targetSetVarElement = element;
                    break;
                }
            }
            currentNode = currentNode.parentNode;
        }

        // 如果找到了变量元素，整体删除
        if (targetVarElement || targetSetVarElement) {
            event.preventDefault();

            const elementToDelete = targetVarElement || targetSetVarElement;
            const tagId = elementToDelete?.getAttribute('tagId');

            if (tagId) {
                // 删除两个相关的 span（target-var 和 target-set-var）
                const relatedElements = editorDiv.value.querySelectorAll(`[tagId="${tagId}"]`);

                // 保存光标位置（在第一个元素之前）
                const firstElement = relatedElements[0];
                const newRange = document.createRange();

                if (firstElement && firstElement.previousSibling) {
                    // 将光标放在删除位置之前
                    if (firstElement.previousSibling.nodeType === Node.TEXT_NODE) {
                        newRange.setStart(firstElement.previousSibling, (firstElement.previousSibling.textContent?.length || 0));
                    } else {
                        newRange.setStartAfter(firstElement.previousSibling);
                    }
                } else if (firstElement && firstElement.parentNode) {
                    newRange.setStart(firstElement.parentNode, 0);
                }

                newRange.collapse(true);

                // 删除所有相关元素
                relatedElements.forEach(el => el.remove());

                // 恢复光标
                selection.removeAllRanges();
                selection.addRange(newRange);

                // 触发 input 事件以更新编辑器状态
                editorDiv.value.dispatchEvent(new Event('input', { bubbles: true }));
            }

            return;
        }
    }

    // 处理输入 { 自动补全 } 并唤起变量浮层
    if (event.key === '{' && !event.ctrlKey && !event.metaKey && !event.altKey) {
        event.preventDefault();

        const selection = window.getSelection();
        if (selection && selection.rangeCount > 0) {
            const range = selection.getRangeAt(0);
            range.deleteContents(); // 删除选中内容（如果有）

            // 插入 {}
            const textNode = document.createTextNode('{}');
            range.insertNode(textNode);

            // 将光标移到 { 和 } 之间
            const newRange = document.createRange();
            newRange.setStart(textNode, 1); // 在第一个字符后（即 { 后面）
            newRange.setEnd(textNode, 1);
            selection.removeAllRanges();
            selection.addRange(newRange);

            // 触发 input 事件以更新编辑器状态
            if (editorDiv.value) {
                editorDiv.value.dispatchEvent(new Event('input', { bubbles: true }));
            }

            // 延迟触发 empty-braces-typed 事件，确保在 processContent 之后
            nextTick(() => {
                emit('empty-braces-typed');
            });
        }
        return;
    }

    if (event.key === 'Enter') {
        const selection = window.getSelection();
        if (selection && selection.rangeCount > 0) {
            let node = selection.anchorNode;
            // 向上查找，检查是否在 .target-var 元素内部
            while (node && node !== editorDiv.value) {
                if (node.nodeType === Node.ELEMENT_NODE && (node as HTMLElement).classList.contains('target-var')) {
                    event.preventDefault(); // 阻止回车键插入换行
                    return;
                }
                node = node.parentNode;
            }


        }
    }

    if (event.ctrlKey || event.metaKey) {
        if (event.key === 'z' && !event.shiftKey) {
            event.preventDefault();
            undo();
        } else if (event.key === 'y' || (event.key === 'z' && event.shiftKey)) {
            event.preventDefault();
            redo();
        }
    }
};



// 编辑器输入事件处理
const handleInput = (e?: Event) => {
    if (isComposing.value || isUndoRedoOperation.value) {
        return;
    }

    const editor = editorDiv.value!;
    // 移除可能存在的 anchor 标签 (用于光标定位)
    editor.querySelectorAll('anchor').forEach(el => el.remove());

    const selection = window.getSelection();
    if (!selection || !selection.rangeCount) return;

    // 插入 anchor 标记当前光标位置
    const range = selection.getRangeAt(0);
    const anchorTag = document.createElement('anchor');
    let shouldEmitEmptyBraces = false;
    try {
        range.insertNode(anchorTag);
        // 检测光标前是否正好是 "{}"，若是则把锚点移动到两括号之间
        const prev = anchorTag.previousSibling;
        if (prev && prev.nodeType === Node.TEXT_NODE) {
            const prevText = (prev.textContent || '').slice(-2);
            if (prevText === '{}') {
                shouldEmitEmptyBraces = true;
                const prevNode = prev as Text;
                const targetOffset = Math.max(0, (prevNode.textContent?.length || 0) - 1);
                const newRange = document.createRange();
                newRange.setStart(prevNode, targetOffset); // 光标定位到 { 和 } 之间
                newRange.collapse(true);
                anchorTag.remove();
                newRange.insertNode(anchorTag);
            }
        }
        range.setStartAfter(anchorTag); // 将光标移到 anchor 之后
        range.setEndAfter(anchorTag);
        selection.removeAllRanges();
        selection.addRange(range);
    } catch (err) {
        console.warn('Error inserting anchor tag, likely due to selection context:', err);
        // 如果插入失败，后续 processContent 中的光标恢复将依赖于 anchorTag 是否存在
    }
    if (shouldEmitEmptyBraces) {
        emit('empty-braces-typed');
    }

    // 处理 target-var 的悬空 }} 问题
    const targetVarsInEditor = editor.querySelectorAll('.target-var');
    targetVarsInEditor.forEach(targetVarEl => {
        if (!(targetVarEl instanceof HTMLElement)) return;
        const tagId = targetVarEl.getAttribute('tagId');
        if (tagId) {
            const correspondingSetVarSpan = editor.querySelector(`.target-set-var[tagId="${tagId}"]`);
            if (!correspondingSetVarSpan) {
                if (targetVarEl.innerText.endsWith('}}')) {
                    let s = targetVarEl.innerHTML;
                    const idx = s.lastIndexOf('}}');
                    if (idx !== -1) {
                        s = s.slice(0, idx + 1) + s.slice(idx + 2);
                    }
                    targetVarEl.innerHTML = s;
                }
            }
        }
    });

    nextTick(() => {

        processContent(); // 处理内容，重新生成变量标签等
        // 历史记录保存推迟，确保在 DOM 更新和变量处理之后
        setTimeout(() => {
            if (!isUndoRedoOperation.value) {
                // 避免在撤销/重做操作中重复保存
                saveToHistory();
            }
        }, 50); // 少量延迟以捕获最终状态
    });
};

// 处理粘贴事件，确保粘贴为纯文本
const handlePaste = (event: ClipboardEvent) => {
    const text = event.clipboardData?.getData('text/plain') || '';

    // 检查粘贴后的总长度，如果超过限制则阻止粘贴
    // if (getContent(false).length + text.length > 4096) {
    //     event.preventDefault();
    //     return false;
    // }

    event.preventDefault();
    document.execCommand('insertText', false, text); // execCommand 已被弃用，但 contenteditable 中仍常用
    // 更好的方式是:
    // const selection = window.getSelection();
    // if (!selection || !selection.rangeCount) return;
    // selection.deleteFromDocument();
    // selection.getRangeAt(0).insertNode(document.createTextNode(text));
    // selection.collapseToEnd();
    // handleInput(); // 手动触发输入处理
};

watch(showPromptParameterEditor, newVal => {
    setTimeout(() => {
        promptEditorRef.value?.clearFormRefValidate();
    }, 10);
});

// 核心内容处理函数：提取变量、更新变量列表、重新渲染编辑器内容
const processContent = () => {
    if (!editorDiv.value || !editorHidden.value) return;

    const currentHtml = editorDiv.value.innerHTML; // 保存处理前的 HTML
    editorHidden.value.innerHTML = currentHtml; // 使用隐藏 div 进行清理操作

    // 从清理后的 HTML 中移除变量控制按钮的 span
    editorHidden.value.querySelectorAll('.target-set-var').forEach(element => {
        element.remove();
    });
    // 将变量占位符的 span 还原为其内部文本，例如 <span class="target-var">{{key}}</span> 变为 {{key}}
    editorHidden.value.querySelectorAll('.target-var').forEach(element => {
        element.outerHTML = element.innerHTML;
    });

    // 重新提取所有变量 key
    const extractedKeys = new Set<string>();
    const tempContentForExtraction = editorHidden.value.innerHTML;
    let match;
    VAR_RE.lastIndex = 0; // 重置正则的 lastIndex

    while ((match = VAR_RE.exec(tempContentForExtraction)) !== null) {
        const rawKey = match[1];
        // 使用临时 div 来剥离 key 中的 HTML 标签，得到纯文本 key
        const tempDivForKey = document.createElement('div');
        tempDivForKey.innerHTML = rawKey;
        const cleanKey = (tempDivForKey.textContent || tempDivForKey.innerText || rawKey).trim();
        if (cleanKey && cleanKey !== '<anchor></anchor>') {
            // 确保 key 不是空的
            extractedKeys.add(cleanKey);
        }
    }

    const oldVariables: Variable[] = JSON.parse(JSON.stringify(variables.value)); // 深拷贝旧变量
    const newVariables: Variable[] = [];

    extractedKeys.forEach(key => {
        const existingVariable = oldVariables.find(v => v.key === key);
        // 根据前缀判断类型：image::/audio::/video:: 为 file，否则为 text-input
        const type = /^(image::|audio::|video::)/.test(key) ? 'file' : 'text-input';

        if (existingVariable) {
            newVariables.push({ key, value: existingVariable.value, type }); // 保留旧值，更新type
        } else {
            newVariables.push({ key, value: '', type }); // 新变量，值为空字符串，设置type
        }
    });
    variables.value = newVariables; // 更新变量列表

    // 基于新的变量列表和提取的 key，重新构建编辑器 HTML
    VAR_RE.lastIndex = 0;
    const newHtml = editorHidden.value.innerHTML.replace(VAR_RE, (fullMatch, keyContent) => {
        const tempDivForKey = document.createElement('div');
        tempDivForKey.innerHTML = keyContent; // keyContent 可能包含 HTML
        const cleanKey = (tempDivForKey.textContent || tempDivForKey.innerText || keyContent).trim();

        if (!cleanKey) return fullMatch; // 如果清理后的 key 为空，则不替换

        const variable = variables.value.find(v => v.key === cleanKey);
        const value = variable ? variable.value : '';
        const hasValue = variable && value !== '';
        const highlightClass = hasValue ? 'highlight-filled' : 'highlight-empty';
        const randomId = Math.random().toString(36).substring(2, 15);
        const editIcon = props.showIcon===true ? '' : editIconSVG;
        const escapedKeyForAttr = escape(cleanKey);
        const escapedAriaLabel = escape(`编辑变量 ${cleanKey}`);
        // 返回包含变量占位符和编辑按钮的 HTML 结构
        return [
            `<span tagId="${randomId}" class="${highlightClass} target-var">{{${keyContent}}}</span>`, // 保留 keyContent 的原始 HTML
            `<span tagId="${randomId}" class="${highlightClass} target-set-var" contenteditable="false" data-key="${escape(cleanKey)}">`,
            `<button class="var-btn" type="button" data-var-key="${escapedKeyForAttr}" aria-label="${escapedAriaLabel}">`,
            `<span class="varValue">${escape(value || '')}</span>`,
            editIcon,
            `</button>`,
            `</span>`,
        ].join('');
    });

    // 只有在 HTML 实际改变时才更新 DOM，以避免不必要的重绘和光标丢失
    if (currentHtml !== newHtml) {
        editorDiv.value.innerHTML = newHtml;
    }

    // 恢复光标位置 (在 anchor 标记之后)
    const anchorEl = editorDiv.value!.querySelector('anchor');
    if (anchorEl) {
        const sel = window.getSelection();
        const range = document.createRange();
        range.setStartAfter(anchorEl);
        range.collapse(true); // 折叠到起始点
        sel?.removeAllRanges();
        sel?.addRange(range);
        anchorEl.remove(); // 移除 anchor 标记
    }

    emitChange(); // 发出 change 事件
};

// 处理编辑器区域点击事件 (用于变量按钮的事件委托)
const handleEditorClick = (event: MouseEvent) => {
    const target = event.target as HTMLElement;
    const varButton = target.closest('.var-btn'); // 寻找最近的 .var-btn 祖先元素

    if (varButton && varButton.hasAttribute('data-var-key')) {
        const key = varButton.getAttribute('data-var-key')!;
        // 点击变量按钮时，显示所有变量的编辑器，并尝试聚焦到被点击的变量
        showVariablesEditor(key);
    }

    // 处理点击外部关闭 PromptParameterEditor 的逻辑已移至全局监听器 handleClickOutside
};

// 点击外部区域关闭 PromptParameterEditor
const handleClickOutside = (event: MouseEvent) => {
    if (!showPromptParameterEditor.value) return;

    const target = event.target as HTMLElement;

    // 检查点击是否在当前组件实例内
    const currentWrapper = document.querySelector(`[data-editor-id="${editorId.value}"]`);
    if (!currentWrapper) return;

    const promptEditorElement = currentWrapper.querySelector('.prompt-editor-single-var');

    // 检查点击是否在当前实例的 PromptParameterEditor 内部
    if (promptEditorElement && !promptEditorElement.contains(target)) {
        // 同时检查点击的是否是当前实例的触发按钮
        const isVarButton = currentWrapper.contains(target) && target.closest('.var-btn');
        const isFillAllButton = fillAllVarsBtn.value && fillAllVarsBtn.value.contains(target);

        if (!isVarButton && !isFillAllButton) {
            promptEditorRef.value?.clearFormRefValidate();
            clickedVarKeyForFocus.value = null;
            showPromptParameterEditor.value = false;
        }
    }

};
// 处理 PromptParameterEditor 的 change 事件
const handlePromptEditorChange = (updatedData: WorkbenchData) => {
    let changed = false;
    // 遍历 PromptParameterEditor 返回的所有数据
    for (const keyInUpdatedData in updatedData) {
        const variable = variables.value.find(v => v.key === keyInUpdatedData);
        if (variable) {
            if (variable.value !== updatedData[keyInUpdatedData]) {
                variable.value = updatedData[keyInUpdatedData]; // 更新变量值
                changed = true;
            }
        } else {
            // 通常不应发生，因为 PromptParameterEditor 是基于当前 variables.value 的 keys 来初始化的
            console.warn(`在 'all' 模式下，尝试更新一个不存在的变量 '${keyInUpdatedData}'。`);
        }
    }

    if (changed) {
        nextTick(() => {
            processContent(); // 重新渲染编辑器内容以反映变量值的变化
            saveToHistory(); // 保存更改后的状态
        });
    }
    // 根据产品需求，可以决定编辑后是否自动关闭 PromptParameterEditor
    // showPromptParameterEditor.value = false;
};

// 处理选区变化，用于显示/隐藏"优化"按钮
const handleSelectionChange = () => {
    if (!editorDiv.value || !optimizeBtn.value) return;
    const selection = window.getSelection();
    if (
        selection &&
        selection.rangeCount > 0 &&
        editorDiv.value.contains(selection.anchorNode) && // 确保选区在编辑器内
        editorDiv.value.contains(selection.focusNode) &&
        !selection.isCollapsed // 确保有选中文本
    ) {
        const range = selection.getRangeAt(0);
        const selectedText = selection.toString().trim();
        let commonAncestor = range.commonAncestorContainer;
        let isInsideVarControl = false; // 检查选区是否在变量控制元素内部

        while (commonAncestor && commonAncestor !== editorDiv.value) {
            if (
                commonAncestor.nodeType === Node.ELEMENT_NODE &&
                ((commonAncestor as HTMLElement).classList.contains('target-set-var') ||
                    (commonAncestor as HTMLElement).classList.contains('target-var'))
            ) {
                isInsideVarControl = true;
                break;
            }
            commonAncestor = commonAncestor.parentNode;
        }

        if (selectedText && !isInsideVarControl) {
            // 有选中文本且不在变量控制元素内
            const rect = range.getBoundingClientRect();
            const editorRect = editorDiv.value.getBoundingClientRect();
            optimizeBtn.value.style.display = 'block';
            optimizeBtn.value.style.top = `${rect.bottom - editorRect.top + editorDiv.value.scrollTop + 5}px`;
            optimizeBtn.value.style.left = `${rect.left - editorRect.left + editorDiv.value.scrollLeft + rect.width / 2 - optimizeBtn.value.offsetWidth / 2}px`;
            return;
        }
    }
    optimizeBtn.value.style.display = 'none'; // 隐藏优化按钮
};

// "优化"按钮点击回调
const onOptimizeClick = () => {
    const selection = window.getSelection();
    if (selection && !selection.isCollapsed) {
        const selectedText = selection.toString().trim();
        if (selectedText) {
            console.log('选中文本（优化点击）：', selectedText);
            // 在此可以添加优化逻辑
        }
    }
    if (optimizeBtn.value) {
        optimizeBtn.value.style.display = 'none';
    }
};

// 处理"AI生成变量"按钮点击 (从 PromptParameterEditor 透传上来)
const handleGenerate = () => {
    // alert('AI 生成变量按钮被点击！');
    // 这里可以调用父组件的AI生成逻辑，或让 PromptParameterEditor 自行处理
    // 如果需要父组件处理，可以在此 emit 一个更具体的事件或调用 props 中的函数
    emit('generate-variables-click');
};
const getContent = (isRendered: boolean): string => {
    if (!editorDiv.value) return '';
    if (isRendered) {
        return contentWithVariablesReplaced();
    } else {
        // 返回纯文本内容，其中变量占位符为 {{key}} 格式
        if (!editorHidden.value) return editorDiv.value.innerText || editorDiv.value.textContent || '';

        if (!editorDiv2.value) return editorDiv.value.innerText || editorDiv.value.textContent || '';
        editorDiv2.value.innerHTML = editorDiv.value.innerHTML;
        // 移除变量编辑按钮相关的span
        editorDiv2.value.querySelectorAll('.target-set-var').forEach(el => el.remove());
        // 将变量span (<span class="target-var">{{key}}</span>) 替换为其内部文本 ({{key}})
        editorDiv2.value.querySelectorAll('.target-var').forEach(el => {
            el.outerHTML = el.innerText;
        });
        // // 将 <br> 替换为换行符以获得更准确的文本表示
        editorDiv2.value.querySelectorAll('br').forEach(br => br.replaceWith('\n'));
        return editorDiv2.value.innerText || '';
    }
};
// --- 暴露给父组件的 API ---
defineExpose({
    openEditorForAllVariables,
    getContent: (isRendered: boolean): string => {
        if (!editorDiv.value) return '';
        if (isRendered) {
            return contentWithVariablesReplaced();
        } else {
            // 返回纯文本内容，其中变量占位符为 {{key}} 格式
            if (!editorHidden.value) return editorDiv.value.innerText || editorDiv.value.textContent || '';

            if (!editorDiv2.value) return editorDiv.value.innerText || editorDiv.value.textContent || '';
            editorDiv2.value.innerHTML = editorDiv.value.innerHTML;
            // 移除变量编辑按钮相关的span
            editorDiv2.value.querySelectorAll('.target-set-var').forEach(el => el.remove());
            // 将变量span (<span class="target-var">{{key}}</span>) 替换为其内部文本 ({{key}})
            editorDiv2.value.querySelectorAll('.target-var').forEach(el => {
                const htmlEl = el as HTMLElement;
                htmlEl.outerHTML = htmlEl.innerText;
            });
            // // 将 <br> 替换为换行符以获得更准确的文本表示
            editorDiv2.value.querySelectorAll('br').forEach(br => br.replaceWith('\n'));
            return editorDiv2.value.innerText || '';
        }
    },
    getVariables: (): Variable[] => {
        return JSON.parse(JSON.stringify(variables.value)); // 返回变量的深拷贝
    },
    validateVariables() {
        return promptEditorRef.value?.validateAll();
    },
    setContent: (content: string): void => {
        promptTxtCopy.value = content; // 更新内部副本
        if (editorDiv.value) {
            // 将传入的纯文本内容（可能包含 {{key}}）设置到编辑器
            // 先将纯文本转换为基础HTML (例如转义特殊字符，但 {{key}} 应该保持原样)
            const tempDiv = document.createElement('div');
            tempDiv.textContent = content;
            editorDiv.value.innerHTML = tempDiv.innerHTML;

            nextTick(() => {
                processContent(); // 处理内容，将 {{key}} 转换为可交互的变量元素
                if (!historyStack.value.length) {
                    // 只有在历史记录为空时（初次设置）才保存
                    saveToHistory(); // 保存初始状态到历史记录
                }
            });
        }
    },
    setVariables: (newVariables: Record<string, string>): void => {
        // 创建现有变量的副本
        const updatedVariables = [...variables.value];

        // 遍历新变量，合并到现有变量中
        Object.entries(newVariables).forEach(([key, value]) => {
            const existingIndex = updatedVariables.findIndex(v => v.key === key);
            // 根据前缀判断类型：image::/audio::/video:: 为 file，否则为 text-input
            const type = /^(image::|audio::|video::)/.test(key) ? 'file' : 'text-input';

            if (existingIndex !== -1) {
                // 如果变量已存在，更新其值和type
                updatedVariables[existingIndex].value = value;
                updatedVariables[existingIndex].type = type;
            } else {
                // 如果变量不存在，添加新变量
                updatedVariables.push({ key, value, type });
            }
        });

        // 更新变量列表
        variables.value = updatedVariables;
        editingParams.value = variablesToWorkbenchData(variables.value);
        nextTick(() => {
            processContent(); // 根据新变量重新渲染编辑器
            saveToHistory(); // 保存状态
        });
        promptEditorRef.value?.closeLoading();
    },
    undo,
    redo,
    canUndo: () => canUndo.value,
    canRedo: () => canRedo.value,
    setEditorFocus,
});

onMounted(() => {
    if (editorDiv.value && !props.promptTxt) {
        // 如果初始 promptTxt 为空或未提供
        // 初始化 processContent 和 saveToHistory，确保空编辑器也有初始状态
        processContent();
        saveToHistory();
    } else if (editorDiv.value && props.promptTxt) {
        // 如果 props.promptTxt 有初始值，watch 的 immediate:true 会调用 setContent，
        // setContent 内部会调用 processContent 和 saveToHistory（如果历史为空）
    }

    document.addEventListener('selectionchange', handleSelectionChange);
    document.addEventListener('click', handleClickOutside, true); // 使用捕获阶段确保能优先处理
});

onUnmounted(() => {
    document.removeEventListener('selectionchange', handleSelectionChange);
    document.removeEventListener('click', handleClickOutside, true);

});
</script>

<style scoped lang="scss">
.editor:empty::before {
    content: '\200B'; /* 零宽空格，确保空编辑器可点击和有高度 */
}

.content-editor-wrapper {
    position: relative; /* 父容器相对定位，以便绝对定位子元素 */
    border: 1px solid #ebedf0;
    border-radius: 8px;
    height: 100%;
    display: flex;
    flex-direction: column;
}

.editor-area {
    min-height: 50px;
    padding: 12px;
    outline: none;
    white-space: pre-wrap; /* 保留空白符序列，但正常换行 */
    word-wrap: break-word; /* 在长单词或 URL 地址内部进行换行 */
    line-height: 22px;
    font-size: 14px;
    font-family: 'PingFang SC';
    color: #252626;
    flex: 1;
    overflow-y: auto;
    max-height: 100%;
    &.editor-area-without-var {
        padding-top: 56px;
    }
    &.editor-disabled {
        cursor: not-allowed;
    }
}

/* 变量占位符 {{key}} 的样式 */
:deep(.target-var) {
    border-radius: 4px 0 0 4px; /* 左侧圆角 */
    padding: 0px 0px 0px 5px; /* 内边距 */
    // background-color: #f0f7ff; /* 默认背景色，由 highlight-empty/filled 控制 */
}

/* 变量编辑按钮区域的样式 */
:deep(.target-set-var) {
    border-radius: 0 4px 4px 0; /* 右侧圆角 */
    // background-color: #f0f7ff; /* 默认背景色 */
    padding: 0px 5px 0px 0px; /* 内边距 */

    button.var-btn {
        /* 按钮特定样式 */
        font-size: 12px;
        background: none;
        border: none;
        padding: 0 2px;
        display: inline-flex; /* 使按钮内元素水平排列 */
        align-items: center; /* 垂直居中对齐 */
        cursor: pointer;
        color: inherit; /* 继承父 span 的颜色 */
        svg {
            transform: translate(0px, -5px); /* 轻微调整SVG图标位置 */
            margin-left: 2px; /* 图标和文字间距 */
        }
    }
}

/* 变量值显示文本的样式 */
:deep(.target-set-var .var-btn .varValue) {
    margin: 0px 2px 0px 2px; /* 调整边距，使其更紧凑 */
    display: inline-block;
    max-width: 120px; /* 限制最大宽度 */
    overflow: hidden;
    text-overflow: ellipsis; /* 超出部分显示省略号 */
    white-space: nowrap; /* 不换行 */
    vertical-align: middle; /* 确保和图标对齐 */
    color: #898a8c;
}

/* 未填写值的变量高亮样式 */
:deep(.highlight-empty.target-var) {
    background-color: #f0f7ff; /* 淡蓝色背景 */
    color: var(--el-text-brand); /* Element Plus 品牌色 */
}
:deep(.highlight-empty.target-set-var) {
    background-color: #f0f7ff;
    color: #30c453;
    button.var-btn svg {
        /* 空状态时图标颜色也调整 */
        fill: #898a8c;
    }
}

/* 已填写值的变量高亮样式 */
:deep(.highlight-filled.target-var) {
    background-color: #eef9f1; /* 淡绿色背景 */
    color: #30c453; /* 深绿色文字 */
}
:deep(.highlight-filled.target-set-var) {
    background-color: #eef9f1;
    color: #2a814b;
    button.var-btn svg {
        /* 填充状态时图标颜色也调整 */
        fill: #898a8c;
    }
    svg {
        transform: translate(0, -2px) !important;
    }
}
.alert-info-wrapper {
    position: absolute;
    background-color: white;
    top: 0;
    left: 12px;
    padding-top: 12px;
    width: calc(100% - 24px);
}
.alert-info {
    width: 100%;
}

.optimize-btn {
    position: absolute;
    background-color: #fff;
    border: 1px solid #ddd;
    border-radius: 4px;
    padding: 5px 10px;
    box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);
    cursor: pointer;
    font-size: 0.9em;
    z-index: 10; /* 确保在编辑器内容之上 */
    transition: opacity 0.1s ease-in-out;
}
.optimize-btn:hover {
    background-color: #f0f0f0;
}

/* PromptParameterEditor (变量编辑弹窗) 的样式 */
.prompt-editor-single-var {
    position: absolute; /* 绝对定位 */
    top: 5px; /* 距离顶部 5px (可调整) */
    right: 5px; /* 距离右侧 5px (可调整) */
    background: white;
    border: 1px solid #e0e0e0;
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15); /* 更明显的阴影 */
    z-index: 1000; /* 确保在最上层 */
    min-width: 336px; /* 最小宽度 */
    border-radius: 6px; /* 圆角 */
}

/* 换行提示样式 */


.editor-toolbar {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 8px 10px;
    border-bottom: 1px solid #e0e0e0;
    background-color: #f8f9fa;
}

.undo-redo-buttons {
    display: flex;
    gap: 8px; /* 按钮间距 */
}

.undo-btn,
.redo-btn,
.editor-toolbar button {
    /* 统一工具栏按钮样式 */
    padding: 4px 8px;
    border: 1px solid #d0d7de;
    border-radius: 4px;
    background-color: #fff;
    cursor: pointer;
    font-size: 12px;
    color: #24292f;
    transition: all 0.2s ease;

    &:hover:not(:disabled) {
        background-color: #f3f4f6;
        border-color: #8b949e;
    }

    &:disabled {
        opacity: 0.6;
        cursor: not-allowed;
        background-color: #f6f8fa;
    }
}
</style>