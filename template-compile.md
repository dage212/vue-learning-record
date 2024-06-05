## template编译（template-compile）

template编译是从compileTemplate函数开始的，这个函数是在vue-plugin插件里面被执行的，options是它的参数，具体类型看typescript，source（template源码），id，filename是必传参数，其它非必传。最终返回编译后js代码

## template编译流程图
<img src="./template-compile.png" alt="template-compile" style="zoom:50%;" />

代码块0
```javascript

export function parse(
  source: string,
  options: SFCParseOptions = {},
): SFCParseResult {
  const sourceKey = genCacheKey(source, options)
  const cache = parseCache.get(sourceKey)
  if (cache) {
    return cache
  }

  const {
    sourceMap = true,
    filename = DEFAULT_FILENAME,
    sourceRoot = '',
    pad = false,
    ignoreEmpty = true,
    compiler = CompilerDOM,
    templateParseOptions = {},
    parseExpressions = true,
  } = options

  const descriptor: SFCDescriptor = {
    filename,
    source,
    template: null,
    script: null,
    scriptSetup: null,
    styles: [],
    customBlocks: [],
    cssVars: [],
    slotted: false,
    shouldForceReload: prevImports => hmrShouldReload(prevImports, descriptor),
  }

  const errors: (CompilerError | SyntaxError)[] = []
  const ast = compiler.parse(source, {
    parseMode: 'sfc',
    prefixIdentifiers: parseExpressions,
    ...templateParseOptions,
    onError: e => {
      errors.push(e)
    },
  })
  ast.children.forEach(node => {
    if (node.type !== NodeTypes.ELEMENT) {
      return
    }
    // we only want to keep the nodes that are not empty
    // (when the tag is not a template)
    if (
      ignoreEmpty &&
      node.tag !== 'template' &&
      isEmpty(node) &&
      !hasSrc(node)
    ) {
      return
    }
    switch (node.tag) {
      case 'template':
        if (!descriptor.template) {
          const templateBlock = (descriptor.template = createBlock(
            node,
            source,
            false,
          ) as SFCTemplateBlock)

          if (!templateBlock.attrs.src) {
            templateBlock.ast = createRoot(node.children, source)
          }

          // warn against 2.x <template functional>
          if (templateBlock.attrs.functional) {
            const err = new SyntaxError(
              `<template functional> is no longer supported in Vue 3, since ` +
                `functional components no longer have significant performance ` +
                `difference from stateful ones. Just use a normal <template> ` +
                `instead.`,
            ) as CompilerError
            err.loc = node.props.find(
              p => p.type === NodeTypes.ATTRIBUTE && p.name === 'functional',
            )!.loc
            errors.push(err)
          }
        } else {
          errors.push(createDuplicateBlockError(node))
        }
        break
      case 'script':
        const scriptBlock = createBlock(node, source, pad) as SFCScriptBlock
        const isSetup = !!scriptBlock.attrs.setup
        if (isSetup && !descriptor.scriptSetup) {
          descriptor.scriptSetup = scriptBlock
          break
        }
        if (!isSetup && !descriptor.script) {
          descriptor.script = scriptBlock
          break
        }
        errors.push(createDuplicateBlockError(node, isSetup))
        break
      case 'style':
        const styleBlock = createBlock(node, source, pad) as SFCStyleBlock
        if (styleBlock.attrs.vars) {
          errors.push(
            new SyntaxError(
              `<style vars> has been replaced by a new proposal: ` +
                `https://github.com/vuejs/rfcs/pull/231`,
            ),
          )
        }
        descriptor.styles.push(styleBlock)
        break
      default:
        descriptor.customBlocks.push(createBlock(node, source, pad))
        break
    }
  })
  if (!descriptor.template && !descriptor.script && !descriptor.scriptSetup) {
    errors.push(
      new SyntaxError(
        `At least one <template> or <script> is required in a single file component.`,
      ),
    )
  }
  if (descriptor.scriptSetup) {
    if (descriptor.scriptSetup.src) {
      errors.push(
        new SyntaxError(
          `<script setup> cannot use the "src" attribute because ` +
            `its syntax will be ambiguous outside of the component.`,
        ),
      )
      descriptor.scriptSetup = null
    }
    if (descriptor.script && descriptor.script.src) {
      errors.push(
        new SyntaxError(
          `<script> cannot use the "src" attribute when <script setup> is ` +
            `also present because they must be processed together.`,
        ),
      )
      descriptor.script = null
    }
  }

  // dedent pug/jade templates
  let templateColumnOffset = 0
  if (
    descriptor.template &&
    (descriptor.template.lang === 'pug' || descriptor.template.lang === 'jade')
  ) {
    ;[descriptor.template.content, templateColumnOffset] = dedent(
      descriptor.template.content,
    )
  }

  if (sourceMap) {
    const genMap = (block: SFCBlock | null, columnOffset = 0) => {
      if (block && !block.src) {
        block.map = generateSourceMap(
          filename,
          source,
          block.content,
          sourceRoot,
          !pad || block.type === 'template' ? block.loc.start.line - 1 : 0,
          columnOffset,
        )
      }
    }
    genMap(descriptor.template, templateColumnOffset)
    genMap(descriptor.script)
    descriptor.styles.forEach(s => genMap(s))
    descriptor.customBlocks.forEach(s => genMap(s))
  }

  // parse CSS vars
  descriptor.cssVars = parseCssVars(descriptor)

  // check if the SFC uses :slotted
  const slottedRE = /(?:::v-|:)slotted\(/
  descriptor.slotted = descriptor.styles.some(
    s => s.scoped && slottedRE.test(s.content),
  )

  const result = {
    descriptor,
    errors,
  }
  parseCache.set(sourceKey, result)
  return result
}

```

代码块1
```javascript
export function compileTemplate(
  options: SFCTemplateCompileOptions,
): SFCTemplateCompileResults {
  const { preprocessLang, preprocessCustomRequire } = options

  const preprocessor = preprocessLang
    ? preprocessCustomRequire
      ? preprocessCustomRequire(preprocessLang)
      : __ESM_BROWSER__
        ? undefined
        : consolidate[preprocessLang as keyof typeof consolidate]
    : false
  if (preprocessor) {
    try {
      return doCompileTemplate({
        ...options,
        source: preprocess(options, preprocessor),
        ast: undefined, // invalidate AST if template goes through preprocessor
      })
    } catch (e: any) {
      return {
        code: `export default function render() {}`,
        source: options.source,
        tips: [],
        errors: [e],
      }
    }
  } else if (preprocessLang) {
    return {
      code: `export default function render() {}`,
      source: options.source,
      tips: [
        `Component ${options.filename} uses lang ${preprocessLang} for template. Please install the language preprocessor.`,
      ],
      errors: [
        `Component ${options.filename} uses lang ${preprocessLang} for template, however it is not installed.`,
      ],
    }
  } else {
    return doCompileTemplate(options)
  }
}
```
代码块2
```javascript
import { 
  createElementVNode as _createElementVNode, 
  resolveComponent as _resolveComponent, 
  createVNode as _createVNode, 
  toDisplayString as _toDisplayString, 
  openBlock as _openBlock, 
  createElementBlock as _createElementBlock, 
  pushScopeId as _pushScopeId, 
  popScopeId as _popScopeId 
} from "vue"
 const _withScopeId = n => (
  _pushScopeId("data-v-4902c357"),
  n=n(),_popScopeId(),
  n
)
const _hoisted_1 = { class: "main" }
const _hoisted_2 = /*#__PURE__*/ _withScopeId(() => /*#__PURE__*/_createElementVNode("p", null, "Main.vue", -1 /* HOISTED */))
export function render(_ctx, _cache, $props, $setup, $data, $options) {
  const _component_module = _resolveComponent("module")
  return (_openBlock(), 
      _createElementBlock(
        "div", 
        _hoisted_1, 
        [
          _hoisted_2,
          _createVNode(_component_module),
          _createElementVNode(
            "code", null, _toDisplayString(_ctx.css), 1 /* TEXT */
          )
        ]
      )
    )
}
```