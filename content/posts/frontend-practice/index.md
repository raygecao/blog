---
weight: 20
title: "React前端快速构建"
subtitle: ""
date: 2024-01-07T12:50:34+08:00
lastmod: 2024-01-07T12:50:34+08:00
draft: false
author: "raygecao"
authorLink: ""
description: ""
resources:
- name: "featured-image"
  src: "featured-image.png"

tags: ["前端"]
categories: ["工作项目"]


toc:
  auto: false
lightgallery: true
license: ""
---
Next.js + NextAuth.js + Material UI + Materio template
<!--more-->

## Next.js

[Next.js](https://nextjs.org/) 是 Vercel 推出的 React Web 框架，提供全栈应用构建的快速解决方案，官方描述其包含 **Routing、Rending、Data Fetching、Styling、Optimizations、Typescripts** 等核心 features。下面根据我们的实践经验描述几个核心特性。

### 全栈式解决方案

官方脚手架构建出的 Next.js 项目包含前后端代码，可以单独启动前/后端服务，也可以同时启动前后端服务。

后端服务使用 Node.js 编写，前端使用 JavaScript/TypeScript 编写，前后端代码在一个项目中管理，避免了前后端分开维护带来的困难。

我们之前为了解决 monorepo 前后端代码跨域访问的问题，做了大量 proxy 及 API endpoints 合并的工作。而该框架使前后端 API 在同一个域内，完美解决跨域访问的问题。

### 客户端渲染 + 服务端渲染

单页应用（SPA）在 React 开发中流行了很长一段时间，此类应用组件渲染完全发生在客户端，使用上更灵活，也更加易于调试，但其存在首屏加载速度慢，不利于 SEO 等问题。

客户端渲染（CSR）模式下，客户端需要请求页面、js/css 文件、动态数据等内容进行渲染，无论是网络请求还是渲染开销都比较大，导致首屏加载速度变慢。另外客户端渲染通常挂一个空的 html 元素，将动态数据渲染到此元素中，从网页源码中无法获取到网站内容，因此不利于 SEO。

服务端渲染（SSR）很好地解决了上述两个问题，服务端将首屏的 html 文件发给客户端，客户端直接展示就可以，这大大减少了首屏加载慢的问题。另外客户端获取到的是静态 html 文件，易于获取网页内容进行 SEO。但其也存在耗费后端资源，与前端耦合度较高的问题。所以**通常情况下首屏加载选择 SSR，之后建议使用CSR**。

CSR 与 SSR 在使用时要注意 JavaScript 与 Node.js 支持的库的差异，比如 React 相关库，浏览器相关 API 只能在客户端渲染使用，filesystem 等只能在服务端渲染中使用。

Next.js 将 CSR 与 SSR 巧妙地结合起来，对于静态内容（如首屏展示内容等）采用SSR，对于用户交互采用CSR。

Next.js 包含两种模式，一种是旧版本的 Page Router 模式，另一种是新版本主推的 App Router 模式。二者使用上最大的差异在于如何声明不同的渲染方式：
- App Router 默认都是服务端渲染，如果需要使用客户端渲染，需要显式指定 `use client`。
- Page Router 通过 `getServerSideProps` 等函数声明服务端渲染所需的请求，函数外部全部为客户端渲染。

### 中间件

Next.js 提供了[中间件](https://nextjs.org/docs/pages/building-your-application/routing/middleware)，对 HTTP Request 进行预处理/后处理。在实践中，我们使用中间件实现了后端鉴权功能，以保证所有到后端的请求都带上 Authorization Header。

另一个比较常见的使用中间件场景是认证。当用户未登录访问安全限制页面时，会跳转到相应的登录页，如 [NextAuth Middleware](https://next-auth.js.org/tutorials/securing-pages-and-api-routes#nextjs-middleware)。

Next.js 中间件最大的坑在于**不支持原生的链式中间件**，我们无法（至少无法很直观地）添加多个中间件以同时支持以上两个预处理功能，真乃大坑也。


### 目录结构的Router

Next.js 支持 App Router 和 Page Router 两种方式，用户可以任选一种进行开发。每一种定义了一组目录路由规则，相比于 `react-dom` 更直观易用。

除了 Page Router 前端页面路由的功能以外，[API Routes](https://nextjs.org/docs/pages/building-your-application/routing/api-routes) 将后端 API path 按照目录结构统一了起来，进一步提升了框架的易用性。


### Proxy

Next.js 支持通过 `next.config.js` 配置 redirects 与 rewrites 实现前端请求代理转发的能力。由于前端请求的 API 可能有一部分是由外部服务（external service）提供的，如果直接访问会出现跨域问题，将对应的 API rewrite 到相应的 external API 可以解决此问题。

使用 proxy 遇到一个坑：当 proxy 一个下载文件的 request 时，由于下载的文件会很大，proxy 不知在哪里加了个 30s timeout，无法使下载顺利完成。翻了翻配置源码找到 `ExperimentalConfig.proxyTimeout`，但并未按预期生效。



## NextAuth.js

[NextAuth.js](https://next-auth.js.org/) 是专门为 Next.js 应用提供认证的工具，具备灵活、易用、安全等特点。后文以 Next.js 的 Page Router 为例说明 NextAuth.js 的用法。

### 添加Providers

通常的认证有两种方式，一种是自行维护登录名和密码的 basic auth 方式，另一种是借助第三方认证系统的 oauth 方式。对于常用的认证方式，NextAuth.js 提供了相应的 provider，因此集成 NextAuth.js 的第一步就是根据特定的方式选择 providers，如选择支持 facebook 与 google 的第三方认证，可以通过以下方法：
```typescript
import NextAuth from 'next-auth'
import FacebookProvider from 'next-auth/providers/facebook'
import GoogleProvider from 'next-auth/providers/google'

export default NextAuth({
  providers: [
    // OAuth authentication providers...
    FacebookProvider({
      clientId: process.env.FACEBOOK_ID,
      clientSecret: process.env.FACEBOOK_SECRET
    }),
    GoogleProvider({
      clientId: process.env.GOOGLE_ID,
      clientSecret: process.env.GOOGLE_SECRET
    }),
  ]
})
```

不同 provider 需要做不同的配置，常见的 provider 分为三类：
- [Oauth Provider](https://next-auth.js.org/configuration/providers/oauth)： 通常需要声明第三方提供的 ID 与 Secret。
- [Email Provider](https://next-auth.js.org/configuration/providers/email)： 通常需要添加 email server。
- [Credentials Provider](https://next-auth.js.org/configuration/providers/credentials)：通常需要配置用户名、密码等用户填入的信息，以及相关的验证逻辑。

### 身份认证

NextAuth.js 支持客户端和服务端的身份认证。

#### 客户端认证

客户端认证使用 [useSession()](https://next-auth.js.org/getting-started/client#usesession) 方法：

```javascript
// 前端组件
import { useSession, signIn, signOut } from "next-auth/react"

export default function Component() {
  const { data: session } = useSession()
  if (session) {
    return (
      <>
        Signed in as {session.user.email} <br />
        <button onClick={() => signOut()}>Sign out</button>
      </>
    )
  }
  return (
    <>
      Not signed in <br />
      <button onClick={() => signIn()}>Sign in</button>
    </>
  )
}
```

使用前需要在 `pages/_app.tsx` 中添加 SessionProvider：
```javascript
import { SessionProvider } from "next-auth/react"
export default function App({
  Component,
  pageProps: { session, ...pageProps },
}) {
  return (
    <SessionProvider session={session}>
      <Component {...pageProps} />
    </SessionProvider>
  )

```

### 服务端认证

服务端认证使用 [getServerSession()](https://next-auth.js.org/configuration/nextjs#getserversession) 方法：

```javascript
import { getServerSession } from "next-auth/next"
import { authOptions } from "./auth/[...nextauth]"

export default async (req, res) => {
  const session = await getServerSession(req, res, authOptions)

  if (session) {
    res.send({
      content:
        "This is protected content. You can access this content because you are signed in.",
    })
  } else {
    res.send({
      error: "You must be signed in to view the protected content on this page.",
    })
  }
}
```
如果使用 JWT 进行认证，服务端认证也可以使用 [getToken()](https://next-auth.js.org/tutorials/securing-pages-and-api-routes#using-gettoken) 方法获取 token 来确认用户身份。

### 前端统一认证跳转

我们应用的场景中，所有前端页面需要用户认证后才可以访问，如果每个页面加一个 `useSession()` 判断，则工作量太大，因此希望可以添加**前端页面未认证自动跳转到登录界面**的功能，上文提到了 NextAuth 中间件可以做到这件事，但我们使用了中间件来添加 Authorization Header。因此选择在 `_app.tsx` 添加一个强制性认证的 HOC：

```typescript
// pages/_app.tsx
const App = (props: ExtendedAppProps) => {
  // ...

  return (    
      <SessionProvider session={session}>
        <Auth>
          <SettingsProvider>
            <SettingsConsumer>
              {({ settings }) => {
                return <ThemeComponent settings={settings}>{getLayout(<Component {...restProps} />)}</ThemeComponent>
              }}
            </SettingsConsumer>
          </SettingsProvider>
        </Auth>
      </SessionProvider>
  )
}

const Auth = ({ children }: { children: React.ReactElement }) => {
  // if `{ required: true }` is supplied, `status` can only be "loading" or "authenticated"
  const { status } = useSession({ required: true })

  if (status === 'loading') {
    return <Loading/>
  }

  return children
}

export default App
```

### 访问外部服务

上文说过 NextAuth.js 在前后端提供了获取解码后的 JWT 的方法，而我们的应用中，大量后端 API 在另一个应用（external backend）中提供，因此我们需要通过 Raw JWT 来与 external backend 交互以验证用户身份。

[getTokn()](https://next-auth.js.org/tutorials/securing-pages-and-api-routes#using-gettoken) 提供了 `raw: true` 属性标识获取Raw JWT。我们可以在访问由 external backend 提供的 API 时，使用 Next.js 中间件添加 Authorization Header 的预处理。

```typescript
// middleware.ts
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'
import { getToken } from 'next-auth/jwt'

const secret = process.env.NEXTAUTH_SECRET

// This function can be marked `async` if using `await` inside
export async function middleware(request: NextRequest) {
  const jwt = await getToken({ req: request, secret: secret, raw: true })
  if (!jwt) {
    return NextResponse.next()
  }

  const requestHeaders = new Headers(request.headers)
  requestHeaders.set('Authorization', `Bearer ${jwt}`)

  return NextResponse.next({
    request: {
      headers: requestHeaders
    }
  })
}

export const config = {
  matcher: '/api/v1/external/:path*'
}
```

由于 NextAuth.js JWT 默认使用 JWE 的编码方式，与 external backend 解析的方式不一致，因此需要[自定义jwt编解码](https://next-auth.js.org/configuration/options#override-jwt-encode-and-decode-methods)：

```typescript
// pages/api/[...nextauth].ts

export const authOptions: AuthOptions = {
    //...
  jwt: {
    encode: async params => {
      const code = await new SignJWT(params.token ?? {})
        .setProtectedHeader({ alg: 'HS256' })
        .setExpirationTime('1d')
        .sign(new TextEncoder().encode(params.secret.toString()))

      return code
    },

    decode: async params => {
      const { token, secret } = params
      if (!token) return null
      const { payload } = await jwtVerify(token, new TextEncoder().encode(secret.toString()))

      return { accessToken: '', ...payload }
    }
  },


```


## Material UI

[Material UI](https://mui.com/material-ui/) 是 Google 推出的 React 组件库，其设计方式简洁扁平，相比 [Ant Design](https://ant-design.antgroup.com/index-cn) 更加轻量，提供更灵活的 CSS 定义方式，适合技术平台的前端开发。

Material UI 的灵魂在于定制化，涵盖 theme，component 等方面。尤其是在定制化组件时，提供了 Base UI, styled components 等手段，为构建组件提供了便捷。

为了快速搭建起一个前端框架，模板是必不可少的，我们使用了开源的 [materio](https://github.com/themeselection/materio-mui-react-nextjs-admin-template-free) 模板，该模板采用 Next.js 12 + Material UI v5 搭建，提供登录、表格、卡片、表单等页面。

### 常用组件

下面记录几个技术平台常用的组件。

#### 代码Diff组件

```typescript
import React from 'react'
import { createPatch } from 'diff'
import * as Diff2Html from 'diff2html'
import 'diff2html/bundles/css/diff2html.min.css'

const CodeDiff: React.FC<{
  fileName: string
  oldStr: string
  newStr: string
  fullContent?: boolean
  oldHeader?: string
  newHeader?: string
}> = ({ fileName, oldStr, newStr, fullContent, oldHeader, newHeader }) => {
  const patch = createPatch(fileName, oldStr, newStr, oldHeader, newHeader, fullContent ? { context: 9999 } : undefined)
  const diffHtml = Diff2Html.html(patch, {
    renderNothingWhenEmpty: true,
    matching: 'lines'
  })

  return <div id='code-diff' dangerouslySetInnerHTML={{ __html: diffHtml }}></div>
}

export default CodeDiff

```

#### Markdown展示组件

```typescript
import React from 'react'
import Typography from '@mui/material/Typography'
import { styled } from '@mui/material/styles'
import Divider from '@mui/material/Divider'
import Link from '@mui/material/Link'
import Checkbox from '@mui/material/Checkbox'
import Table from '@mui/material/Table'
import TableBody from '@mui/material/TableBody'
import TableCell from '@mui/material/TableCell'
import TableContainer from '@mui/material/TableContainer'
import TableHead from '@mui/material/TableHead'
import TableRow from '@mui/material/TableRow'
import ReactMarkdown from 'marked-react'
import CodeBoard from './CodeBoard'
import Paper from '@mui/material/Paper'

const Wrapper = styled('div')({
  '& .MuiTypography-h4': {
    marginTop: '0.2em'
  },
  '& .MuiTypography-h5': {
    marginTop: '1.5em',
    marginBottom: '1em'
  },
  '& .MuiTypography-h6': {
    marginTop: '1.5em',
    marginBottom: '1em'
  },
  '& p': {
    marginBottom: '0.4em'
  },
  '& .MuiDivider-root': {
    marginBottom: '0.4em'
  },
  '& code': {
    fontSize: '0.9em'
  }
})

const genKey = (e: unknown): string => {
  const ee = e as { elementId: number }

  return `markdown-${ee.elementId}`
}

const fixLang = (lang?: string): string => {
  if (!lang) {
    return 'text'
  }

  return lang.split(' ')[0]
}

const renderer: () => Parameters<typeof ReactMarkdown>[0]['renderer'] = () => ({
  code(snippet, lang) {
    return (
      <Paper key={genKey(this)} variant='outlined' sx={{ m: 1 }}>
        <CodeBoard code={snippet as string} language={fixLang(lang)} title={''} />
      </Paper>
    )
  },
  link(href, text) {
    return (
      <Link key={genKey(this)} target='_blank' rel='noopener' href={href}>
        {text}
      </Link>
    )
  },
  paragraph(children) {
    return (
      <Typography key={genKey(this)} variant='body1' gutterBottom={false} paragraph={true}>
        {children}
      </Typography>
    )
  },
  heading(children, level) {
    switch (level) {
      case 1:
        return (
          <Typography key={genKey(this)} variant='h4' gutterBottom={true} component='h1'>
            {children}
          </Typography>
        )
      case 2:
        return (
          <Typography key={genKey(this)} variant='h5' gutterBottom={true} component='h2'>
            {children}
          </Typography>
        )
      case 3:
        return (
          <Typography key={genKey(this)} variant='h6' gutterBottom={true} component='h3'>
            {children}
          </Typography>
        )
      case 4:
        return (
          <Typography key={genKey(this)} variant='subtitle1' gutterBottom={true} component='h4'>
            {children}
          </Typography>
        )
      case 5:
        return (
          <Typography key={genKey(this)} variant='subtitle2' gutterBottom={true} component='h5'>
            {children}
          </Typography>
        )
      default:
        return (
          <Typography key={genKey(this)} variant='caption' gutterBottom={true} component='h6'>
            {children}
          </Typography>
        )
    }
  },
  image(href, text, title) {
    return (
      <img
        key={genKey(this)}
        src={href}
        alt={text}
        title={title === null ? undefined : title}
        style={{ maxWidth: '100%' }}
      />
    )
  },
  listItem(children) {
    return (
      <Typography key={genKey(this)} gutterBottom={false} variant='body1' component='li'>
        {children}
      </Typography>
    )
  },
  checkbox(checked) {
    return <Checkbox key={genKey(this)} size='small' disabled={true} sx={{ p: 0 }} checked={checked as boolean} />
  },
  table(children) {
    return (
      <TableContainer key={genKey(this)}>
        <Table size='small' sx={{ display: 'inline-block' }}>
          {children}
        </Table>
      </TableContainer>
    )
  },
  tableHeader(children) {
    return <TableHead key={genKey(this)}>{children}</TableHead>
  },
  tableBody(children) {
    return <TableBody key={genKey(this)}>{children}</TableBody>
  },
  tableRow(children) {
    return <TableRow key={genKey(this)}>{children}</TableRow>
  },
  tableCell(children) {
    return <TableCell key={genKey(this)}>{children}</TableCell>
  },
  hr() {
    return <Divider key={genKey(this)} />
  }
})

const Markdown: React.FC<{ text: string }> = props => {
  return (
    <Wrapper>
      <ReactMarkdown value={props.text} renderer={renderer()} />
    </Wrapper>
  )
}
export default Markdown

```

#### 搜索框

```typescript
import React, { useEffect, useState } from 'react'
import { styled } from '@mui/material/styles'
import InputBase from '@mui/material/InputBase'
import SearchIcon from '@mui/icons-material/Search'
import { useRouter } from 'next/router'

const Search = styled('div')(({ theme }) => ({
  position: 'relative',
  borderRadius: theme.shape.borderRadius,
  backgroundColor: theme.palette.mode === 'dark' ? 'rgba(255,255,255,0.1)' : 'rgba(0,0,0,0.07)',

  marginLeft: 0,
  marginRight: theme.spacing(2),
  width: 'auto',
  [theme.breakpoints.up('sm')]: {
    marginRight: theme.spacing(1),
    marginLeft: theme.spacing(1),
    width: 'auto'
  }
}))

const SearchIconWrapper = styled('div')(({ theme }) => ({
  padding: theme.spacing(0, 2),
  height: '100%',
  position: 'absolute',
  pointerEvents: 'none',
  display: 'flex',
  alignItems: 'center',
  justifyContent: 'center'
}))

const StyledInputBase = styled(InputBase)(({ theme }) => ({
  color: 'inherit',
  '& .MuiInputBase-input': {
    padding: theme.spacing(1, 1, 1, 0),

    // vertical padding + font size from searchIcon
    paddingLeft: `calc(1em + ${theme.spacing(4)})`,

    transition: theme.transitions.create(['width', 'opacity']),
    width: '12ch',
    opacity: 0.4
  },
  '& .MuiInputBase-input:focus': {
    width: '20ch',
    opacity: 1
  }
}))

const EmbeddedStyledInputBase = styled(InputBase)(({ theme }) => ({
  color: 'inherit',
  '& .MuiInputBase-input': {
    padding: theme.spacing(1, 1, 1, 0),

    // vertical padding + font size from searchIcon
    paddingLeft: `calc(1em + ${theme.spacing(4)})`,

    // transition: theme.transitions.create(['width', 'opacity']),
    width: '12ch',
    opacity: 0.4
  }
}))

const SearchBar: React.FC<{
  onSearch: (_: string) => void
  placeholder: string
  value?: string
  embedded?: boolean
}> = props => {
  const [search, setSearch] = useState<null | string>(null)

  const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault()
    e.stopPropagation()
    props.onSearch(search === null ? props.value ?? '' : search)
  }
  const handleSearchChange = (evt: React.ChangeEvent<HTMLInputElement>) => {
    setSearch(evt.target.value)
  }

  useEffect(() => {
    setSearch(props.value ?? '')
  }, [props.value])

  const IBase = props.embedded ? EmbeddedStyledInputBase : StyledInputBase

  return (
    <Search>
      <SearchIconWrapper>
        <SearchIcon />
      </SearchIconWrapper>
      <form onSubmit={handleSubmit}>
        <IBase placeholder={props.placeholder} value={search ?? props.value} onChange={handleSearchChange} />
      </form>
    </Search>
  )
}
export default SearchBar
```

#### 确认对话框

```typescript
import Dialog from '@mui/material/Dialog'
import DialogTitle from '@mui/material/DialogTitle'
import DialogContent from '@mui/material/DialogContent'
import Typography from '@mui/material/Typography'
import DialogActions from '@mui/material/DialogActions'
import Button from '@mui/material/Button'

type confirmDialogOpts = {
  open: boolean
  title: string
  content: string
  onConfirm: () => void
  onCancel: () => void
}

const ConfirmDialog = ({ open, title, content, onConfirm, onCancel }: confirmDialogOpts) => {
  return (
    <Dialog open={open} fullWidth onClose={onCancel}>
      <DialogTitle>{title}</DialogTitle>
      <DialogContent>
        <Typography>{content}</Typography>
      </DialogContent>
      <DialogActions>
        <Button variant='outlined' onClick={onCancel}>
          取消
        </Button>
        <Button variant='contained' onClick={onConfirm}>
          确认
        </Button>
      </DialogActions>
    </Dialog>
  )
}

export default ConfirmDialog
```


#### 表单

表单的设计采用 [react-hook-form](https://react-hook-form.com/)，并使用 [zod](https://zod.dev/) 对用户填写的内容进行校验。示例中较为麻烦的是基于 AutoComplete 的多选输入。

```typescript
import { useEffect, useMemo } from 'react'
import { useForm, SubmitHandler, Controller } from 'react-hook-form'
import Button from '@mui/material/Button'
import TextField from '@mui/material/TextField'
import Box from '@mui/material/Box'
import Grid from '@mui/material/Grid'
import Typography from '@mui/material/Typography'
import Autocomplete from '@mui/material/Autocomplete'
import { z } from 'zod'
import { zodResolver } from '@hookform/resolvers/zod'

type CatalogForm = {
  onSubmit: SubmitHandler<composeForm>
  handleClose: () => void
  plugins: string[]
  version: string
  name?: string
}
export const composeFormSchema = z.object({
  name: z.string().min(3, { message: '名称长度至少为3' }),
  description: z.string(),
  plugins: z.array(z.string()).min(1, { message: '须选择至少一个插件' }),
  version: z
    .string()
    .regex(
      /^(0|[1-9]\d*)\.(0|[1-9]\d*)\.(0|[1-9]\d*)(?:-((?:0|[1-9]\d*|\d*[a-zA-Z-][0-9a-zA-Z-]*)(?:\.(?:0|[1-9]\d*|\d*[a-zA-Z-][0-9a-zA-Z-]*))*))?(?:\+([0-9a-zA-Z-]+(?:\.[0-9a-zA-Z-]+)*))?$/,
      'version must meet the format of semver2'
    )
})

export type composeForm = z.infer<typeof composeFormSchema>

const ComposeForm = ({ onSubmit, handleClose, plugins, version, name }: CatalogForm) => {
  const defaultValues = useMemo(
    () => ({
      description: '',
      name: name ?? 'default',
      version: version,
      plugins: []
    }),
    [version, name]
  )

  const {
    control,
    handleSubmit,
    reset,
    formState: { errors, isSubmitSuccessful }
  } = useForm<composeForm>({
    defaultValues: defaultValues,
    resolver: zodResolver(composeFormSchema)
  })

  useEffect(() => {
    if (isSubmitSuccessful) {
      reset()
    }
  }, [isSubmitSuccessful, reset])

  useEffect(() => {
    reset(defaultValues)
  }, [defaultValues, reset])

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <Grid container spacing={5}>
        <Grid item xs={12}>
          <Typography variant='body2' sx={{ fontWeight: 600 }}>
            选择插件
          </Typography>
        </Grid>
        <Grid item xs={12} sm={6}>
          <Controller
            name={`plugins`}
            control={control}
            render={({ field }) => (
              <Autocomplete
                {...field}
                multiple
                renderInput={params => (
                  <TextField
                    {...params}
                    {...field}
                    label='选择插件'
                    error={!!errors.plugins}
                    helperText={errors.plugins?.message ?? ''}
                  />
                )}
                options={plugins}
                onChange={(_, data) => field.onChange(data ?? '')}
              ></Autocomplete>
            )}
          />
        </Grid>
        <Grid item xs={12} sm={6}>
          <Controller
            name='name'
            control={control}
            render={({ field }) => (
              <TextField
                {...field}
                label='名称'
                fullWidth
                error={!!errors.name}
                helperText={errors.name?.message ?? ''}
                required
              />
            )}
          />
        </Grid>
        <Grid item xs={12} sm={6}>
          <Controller
            name='version'
            control={control}
            render={({ field }) => (
              <TextField
                {...field}
                label='版本'
                fullWidth
                error={!!errors.version}
                helperText={errors.version?.message ?? ''}
                required
              />
            )}
          />
        </Grid>
        <Grid item xs={12}>
          <Controller
            name='description'
            control={control}
            render={({ field }) => (
              <TextField
                {...field}
                label={'描述'}
                fullWidth
                error={!!errors.description}
                helperText={errors.description?.message ?? ''}
              />
            )}
          />
        </Grid>
      </Grid>
      <Box sx={{ display: 'flex', justifyContent: 'flex-end', gap: 4, mt: 4 }}>
        <Button variant='outlined' onClick={handleClose}>
          取消
        </Button>
        <Button type='submit' variant='contained'>
          确定
        </Button>
      </Box>
    </form>
  )
}

export default ComposeForm
```

如果表单内容较多，在对排版要求不严格的情况下，建议使用 [react-jsonschema-form](https://github.com/rjsf-team/react-jsonschema-form) ，通过 json schema 自动创建表单。

#### 通知栏

使用 React Context 传递更改状态栏的方法，各个组件可以使用该方法显示状态栏。

1. 定义 Notification Context 和通知栏组件：

```typescript
import React, { createContext, useContext, useState } from 'react'
import Snackbar from '@mui/material/Snackbar'
import Alert, { AlertColor } from '@mui/material/Alert'

type Notification = {
  severity: AlertColor
  message: string
  duration?: number
}

const NotificationCtx = createContext<{
  notification: Notification | null
  setNotification: (n: Notification) => void
  hideNotification: () => void
}>({
  notification: null,
  setNotification: (_: Notification) => null,
  hideNotification: () => null,
})

export const NotificationCtxProvider: React.FC<{
  children?: React.ReactNode
}> = (props) => {
  const [notification, setNotification] = useState<Notification | null>(null)
  const context = {
    notification: notification,
    setNotification: (n: Notification) => setNotification(n),
    hideNotification: () => setNotification(null),
  }
  return (
    <NotificationCtx.Provider value={context}>
      {props.children}
    </NotificationCtx.Provider>
  )
}

export const NotificationComp: React.FC = () => {
  const ctx = useContext(NotificationCtx)
  return (
    <Snackbar
      sx={{ maxWidth: '400px' }}
      open={!!ctx.notification}
      autoHideDuration={ctx.notification?.duration ?? 5000}
      onClose={ctx.hideNotification}
      message={ctx.notification?.message}
      anchorOrigin={{ vertical: 'top', horizontal: 'right' }}
    >
      <Alert
        onClose={ctx.hideNotification}
        severity={ctx.notification?.severity}
      >
        {ctx.notification?.message}
      </Alert>
    </Snackbar>
  )
}

export default NotificationCtx
```

2. 将通知栏组件挂在顶层页面上：

```typescript
  <Router>
    <ThemeProvider locale={i18n.language} themeOptions={makeTheme}>
      <Box sx={{ display: 'flex' }}>
        <Nav />
        <Main />
        <NotificationComp />
      </Box>
    </ThemeProvider>
  </Router>
```

3. 相关组件从 context 中获取更改状态栏的函数并调用：

```typescript=
export const SomeComponent = () => {

  const ctx = useContext(NotificationCtx)

  return (
    <Button
      onClick={() =>
        ctx.setNotification({
          severity: 'error',
          message: 'some message'
        })
      }
    >
      Click me
    </Button>
  )
}

```

## 踩坑记录

- Next.js proxy 存在30s超时时间的设置，且不易溯源。
- Next.js 除了通过环境变量文件声明，后端服务也可以直接使用系统环境变量，如果前端需要获取某个敏感变量时，可以借助后端 API Routes 传递。
- NextAuth.js 自定义登录页时，Auth HoC 会引起页面循环重定向。
- 页面切换后，react-hook-form 需要显式执行 `reset()`，才能使得基于 state 的表单默认值发生变化。
- 不要将组件的 props 作为组件内 state 的 initial value，否则 props 变化后，state 不会发生变化。
- React hook 使用之前不能有判断语句，state/props 使用上不符合预期时，优先考虑 `useEffect()` 能否解决。

## References

- [hydration in React](https://medium.com/@gautamkiran123/hydration-in-react-8e8dff384f93)
- [React Antipatterns: Props In Initial State](https://vhudyma-blog.eu/react-antipatterns-props-in-initial-state/)
- [NextAuth Pages](https://next-auth.js.org/configuration/pages#examples)
- [Infinite redirection loop with <Auth /> wrapper when using custom sign-in page](https://github.com/nextauthjs/next-auth/issues/4340)
