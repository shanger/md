# vue css scoped 的实现
## 背景
使用scoped的初衷都一样，为了组件内的样式不受外部影响，组件内的样式不会干扰其他元素。我们有一个场景，业务组件引用了基础组件，我们希望在业务组件内对基础组件的样式进行控制但是结果不尽人意，业务组件内基础组件并没有被添加上特殊ID导致样式无效。一时没有找到解决方案的我们，选择通过类似bem的形式解决了我们的问题。但是Vue的scoped是怎么实现，为什么组件内的元素没有被打上标记，我们尝试了解其中的实现过程。
## 问题分析
* 期望
在父组件内使用scoped，控制组件内所有样式。
* 结果
子组件内元素没有被标记，导致生成的css无法匹配子组件内元素，样式设置无效。
* 分析目的
了解为什么子组件内的元素不会被特殊标记，仅仅标记了子组件的根节点的设计初衷，以及实现过程。
## vue loader
单文件组件的引入是通过vueloader来处理的，我们来看看vueloader都做了什么
1. 解析源文件，分类处理交给下一步loader
```
const descriptor = parse({
  source,
  compiler: options.compiler || loadTemplateCompiler(loaderContext),
  filename,
  sourceRoot,
  needMap: sourceMap
})
const id = hash(
  isProduction
    ? (shortFilePath + '\n' + source)
    : shortFilePath
)

// scoped应用
const hasScoped = descriptor.styles.some(s => s.scoped)

// template
  let templateImport = `var render, staticRenderFns`
  let templateRequest
  if (descriptor.template) {
    const src = descriptor.template.src || resourcePath
    const idQuery = `&id=${id}`
    const scopedQuery = hasScoped ? `&scoped=true` : ``
    const attrsQuery = attrsToQuery(descriptor.template.attrs)
    const query = `?vue&type=template${idQuery}${scopedQuery}${attrsQuery}${inheritQuery}`
    const request = templateRequest = stringifyRequest(src + query)
    templateImport = `import { render, staticRenderFns } from ${request}`
    // import { render, staticRenderFns } from ./src/form10-components/components/duowanComponents/Jika.vue?vue&type=template&id=8c52d2d2&
  }

  // script
  let scriptImport = `var script = {}`
  if (descriptor.script) {
    const src = descriptor.script.src || resourcePath
    const attrsQuery = attrsToQuery(descriptor.script.attrs, 'js')
    const query = `?vue&type=script${attrsQuery}${inheritQuery}`
    const request = stringifyRequest(src + query)
    scriptImport = (
      `import script from ${request}\n` +
      `export * from ${request}` // support named exports
    )
  }

  // styles
  let stylesCode = ``
  if (descriptor.styles.length) {
    stylesCode = genStylesCode(
      loaderContext,
      descriptor.styles,
      id,
      resourcePath,
      stringifyRequest,
      needsHotReload,
      isServer || isShadow // needs explicit injection?
    )
  }
```
第一步先将文件按template、script、style分开处理，拼接处对应的query路径，然后交给webpack的对应loader继续处理。
2. Loader

template类型的交由templateLoader处理
```
const templateLoaderPath = require.resolve('./templateLoader')
if (query.type === `template`) {
  const path = require('path')
  // ...
  const request = genRequest([
    templateLoaderPath + `??vue-loader-options`,
    //...
  ])
  // the template compiler uses esm exports
  return `export * from ${request}`
}
```
templateLoader主要用于模板的预编译，引用的是`vue-template-compiler`
```
const { compileTemplate } = require('@vue/component-compiler-utils')


module.exports = function (source) {
  const loaderContext = this
  const query = qs.parse(this.resourceQuery.slice(1))
  const options = loaderUtils.getOptions(loaderContext) || {}
  const { id } = query
  const isServer = loaderContext.target === 'node'
  const isProduction = options.productionMode || loaderContext.minimize || process.env.NODE_ENV === 'production'
  const isFunctional = query.functional

  // 允许自定义compiler，没有的话用的就是Vue package提供的库
  const compiler = options.compiler || require('vue-template-compiler')

  const compilerOptions = Object.assign({
    outputSourceRange: true
  }, options.compilerOptions, {
    scopeId: query.scoped ? `data-v-${id}` : null,
    comments: query.comments
  })

  // for vue-component-compiler
  const finalOptions = {
    source,
    filename: this.resourcePath,
    compiler,
    compilerOptions,
    // allow customizing behavior of vue-template-es2015-compiler
    transpileOptions: options.transpileOptions,
    transformAssetUrls: options.transformAssetUrls || true,
    isProduction,
    isFunctional,
    optimizeSSR: isServer && options.optimizeSSR !== false,
    prettify: options.prettify
  }

  const compiled = compileTemplate(finalOptions)

  // tips
  if (compiled.tips && compiled.tips.length) {
    compiled.tips.forEach(tip => {
      loaderContext.emitWarning(typeof tip === 'object' ? tip.msg : tip)
    })
  }
  // ...

  const { code } = compiled

  // finish with ESM exports
  return code + `\nexport { render, staticRenderFns }`
}
```
至此我们已经知道，vueloader对单文件组件的加载和编译过程，以及scopeid的生成过程，但具体在render的过程中是怎么做的，为什么不向children传入，我们继续分析。
3. compile细节

```
function setScope (vnode) {
    let i
    if (isDef(i = vnode.fnScopeId)) {
      nodeOps.setStyleScope(vnode.elm, i)
    } else {
      let ancestor = vnode
      // 向父级查找有没有scopeId
      while (ancestor) {
        if (isDef(i = ancestor.context) && isDef(i = i.$options._scopeId)) {
          nodeOps.setStyleScope(vnode.elm, i)
        }
        ancestor = ancestor.parent
      }
    }
    // for slot content they should also get the scopeId from the host instance.
    if (isDef(i = activeInstance) &&
      i !== vnode.context &&
      i !== vnode.fnContext &&
      isDef(i = i.$options._scopeId)
    ) {
      nodeOps.setStyleScope(vnode.elm, i)
    }
  }
```
根节点_vnode是有parent的，但是其他children的节点找不到。