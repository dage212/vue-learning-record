doCompileTemplate主要是生成编译好的模板代码，生成sourcemap，生成ast
```javascript
function doCompileTemplate({
  filename,
  id,
  scoped,
  slotted,
  inMap,
  source,
  ast: inAST,
  ssr = false,
  ssrCssVars,
  isProd = false,
  compiler,
  compilerOptions = {},
  transformAssetUrls,
}: SFCTemplateCompileOptions): SFCTemplateCompileResults {
  const errors: CompilerError[] = []
  const warnings: CompilerError[] = []

  let nodeTransforms: NodeTransform[] = []
  //createAssetUrlTransformWithOptions这个插件用来处理img的src的url
  //createSrcsetTransformWithOptions用来处理img的srcset的url
  //例如：<img src="small.jpg " srcset="big.jpg 1440w, middle.jpg 800w, small.jpg 1x" />
  if (isObject(transformAssetUrls)) {
    const assetOptions = normalizeOptions(transformAssetUrls)
    nodeTransforms = [
      createAssetUrlTransformWithOptions(assetOptions),
      createSrcsetTransformWithOptions(assetOptions),
    ]
  } else if (transformAssetUrls !== false) {
    nodeTransforms = [transformAssetUrl, transformSrcset]
  }

  if (ssr && !ssrCssVars) {
    warnOnce(
      `compileTemplate is called with \`ssr: true\` but no ` +
        `corresponding \`cssVars\` option.\`.`,
    )
  }
  if (!id) {
    warnOnce(`compileTemplate now requires the \`id\` option.\`.`)
    id = ''
  }

  const shortId = id.replace(/^data-v-/, '')
  const longId = `data-v-${shortId}`
  //如果ssr为true，则使用ssr编译器(compiler-ssr里面的complier)否则用默认的编译器(compiler里面的complier)
  
  const defaultCompiler = ssr ? (CompilerSSR as TemplateCompiler) : CompilerDOM
  //这里的compiler是从vue-plugin的buildStart里面赋值的
  compiler = compiler || defaultCompiler

  if (compiler !== defaultCompiler) { //用自定义compiler就不能用默认生成的ast
    // user using custom compiler, this means we cannot reuse the AST from
    // the descriptor as they might be different.
    inAST = undefined
  }
  //这里的inAST是descriptor里面的template里面的ast,descriptor这个对象记录了<template> 、<script>和<style>标签的信息
  if (inAST?.transformed) {
    // If input AST has already been transformed, then it cannot be reused.
    // We need to parse a fresh one. Can't just use `source` here since we need
    // the AST location info to be relative to the entire SFC.
    //inAST.source是组件源码包含<template>\ <script>和<style>内容
    const newAST = (ssr ? CompilerDOM : compiler).parse(inAST.source, {
      prefixIdentifiers: true,
      ...compilerOptions,
      parseMode: 'sfc',
      onError: e => errors.push(e),
    })
    const template = newAST.children.find(
      node => node.type === NodeTypes.ELEMENT && node.tag === 'template',
    ) as ElementNode
    inAST = createRoot(template.children, inAST.source)
  }
  //这里最终返回编译成代码块2的代码,生成ast和sourcemap  
  let { code, ast, preamble, map } = compiler.compile(inAST || source, {
    mode: 'module',
    prefixIdentifiers: true,
    hoistStatic: true,
    cacheHandlers: true,
    ssrCssVars:
      ssr && ssrCssVars && ssrCssVars.length
        ? genCssVarsFromList(ssrCssVars, shortId, isProd, true)
        : '',
    scopeId: scoped ? longId : undefined,
    slotted,
    sourceMap: true,
    ...compilerOptions,
    hmr: !isProd,
    nodeTransforms: nodeTransforms.concat(compilerOptions.nodeTransforms || []),
    filename,
    onError: e => errors.push(e),
    onWarn: w => warnings.push(w),
  })

  // inMap should be the map produced by ./parse.ts which is a simple line-only
  // mapping. If it is present, we need to adjust the final map and errors to
  // reflect the original line numbers.
  if (inMap && !inAST) {
    if (map) {
      map = mapLines(inMap, map)
    }
    if (errors.length) {
      patchErrors(errors, source, inMap)
    }
  }

  const tips = warnings.map(w => {
    let msg = w.message
    if (w.loc) {
      msg += `\n${generateCodeFrame(
        inAST?.source || source,
        w.loc.start.offset,
        w.loc.end.offset,
      )}`
    }
    return msg
  })

  return { code, ast, preamble, source, errors, tips, map }
}
```