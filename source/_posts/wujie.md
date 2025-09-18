---
title: 无界问题
---

## 1.css 样式内部的相对地址相对的是主应用的域名
在本地开发时，css 样式内部的相对地址相对的是主应用的域名，可以使用Vite Plugin或者 Webpack loader来进行处理
如下代码实例，使用正则匹配
``` bash
import type { Plugin } from 'vite'

// 动态获取基础 URL 的函数
const getBaseUrl = (baseUrl?: string, mode?: string) => {
    // 如果传入了 baseUrl，直接使用
    if (baseUrl) {
        return baseUrl
    }
    
    // 根据环境变量和模式动态获取
    const isProduction = process.env.NODE_ENV === 'production'
    const isDev = mode === 'development' || !isProduction
    
    if (isDev) {
        // 开发环境：优先使用传入的端口号，然后从环境变量获取
        const devPort = process.env.VITE_DEV_PORT || process.env.PORT || 5183
        return `http://localhost:${devPort}`
    } else {
        // 生产环境：从环境变量获取或使用默认值
        return process.env.VITE_PROD_BASE_URL || 
               process.env.VITE_CSS_BASE_URL || 
               'https://your-production-domain.com'
    }
}

// CSS 拦截插件：将相对地址转换为绝对地址
export const cssUrlTransformPlugin = (baseUrl?: string, mode?: string, debug?: boolean): Plugin => {
    const isDebug = debug;
    
    const debugLog = (...args: any[]) => {
        if (isDebug) {
            console.log(...args)
        }
    }
    
    debugLog(`[CSS URL Transform] Plugin initialized with baseUrl: ${baseUrl}, mode: ${mode}, debug: ${isDebug}`)
    
    return {
        name: 'css-url-transform',
        transform(code, id) {
            // 在线上环境不仅入下列处理流程
            if (mode === 'production') {
                return null
            }
            // 只处理 CSS 和 SCSS 文件
            if (!id.endsWith('.css') && !id.endsWith('.scss') && !id.endsWith('.vue')) {
                return null
            }
            
            // 获取动态基础 URL
            const finalBaseUrl = getBaseUrl(baseUrl, mode)
            
            debugLog(`[CSS URL Transform] Processing file: ${id}`)
            debugLog(`[CSS URL Transform] Using base URL: ${finalBaseUrl}`)
            
            // 匹配 url() 中的相对路径
            const urlRegex = /url\(['"]?([^'")]+)['"]?\)/g
            
            // 先检查是否有匹配的 URL
            const matches = code.match(urlRegex)
            debugLog(`[CSS URL Transform] Found ${matches ? matches.length : 0} URL matches in ${id}`)
            if (matches) {
                debugLog(`[CSS URL Transform] Matches:`, matches)
            }
            
            const transformedCode = code.replace(urlRegex, (match, url) => {
                debugLog(`[CSS URL Transform] Processing URL: ${url}`)
                
                // 跳过已经是绝对路径的 URL
                if (url.startsWith('http://') || url.startsWith('https://') || url.startsWith('data:')) {
                    debugLog(`[CSS URL Transform] Skipping absolute URL: ${url}`)
                    return match
                }
              
                // 跳过以 / 开头的绝对路径
                if (url.startsWith('/')) {
                    debugLog(`[CSS URL Transform] Skipping absolute path: ${url}`)
                    return match
                }
              
                // 跳过以 # 开头的锚点链接
                if (url.startsWith('#')) {
                    debugLog(`[CSS URL Transform] Skipping anchor link: ${url}`)
                    return match
                }
              
                // 处理 ~@ 路径别名
                let processedUrl = url
                if (processedUrl.startsWith('~@/')) {
                    processedUrl = processedUrl.replace('~@/', 'src/')
                    debugLog(`[CSS URL Transform] Converted ~@ to src: ${url} -> ${processedUrl}`)
                }
              
                // 确保 baseUrl 不以 / 结尾，url 不以 / 开头
                const cleanBaseUrl = finalBaseUrl.replace(/\/$/, '')
                const cleanUrl = processedUrl.replace(/^\//, '')
              
                // 去掉路径中的 creative-center
                const finalUrl = cleanUrl.replace(/^creative-center\//, '')
              
                const absoluteUrl = `${cleanBaseUrl}/${finalUrl}`
                debugLog(`[CSS URL Transform] ${url} -> ${absoluteUrl}`)
                return `url('${absoluteUrl}')`
            })
            
            // 如果代码有变化，返回转换后的代码
            if (transformedCode !== code) {
                debugLog(`[CSS URL Transform] Code transformed for ${id}`)
                return {
                    code: transformedCode,
                    map: null
                }
            }
            
            return null
        }
    }
}
```
