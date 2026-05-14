---
title: "Ant Design (antd) — B2B 풍 컴포넌트"
kind: knowledge
project: frontend
agent: engineering-agent/tech-lead
status: current
tags: [frontend, react, ui-library, antd, ant-design]
---

# Ant Design (antd)

**[[ui-libraries|↑ UI Libraries Hub]]**

> `masterway-dev/job-answer-fe` 사용. v5 기준.

## 1. 설치

```bash
yarn add antd
yarn add @ant-design/icons
```

## 2. import + reset

```tsx
// main.tsx
import 'antd/dist/reset.css';
```

→ v5 부터는 CSS 안 import 해도 자동 styling 가능 (CSS-in-JS). 그래도 reset 권장.

## 3. ConfigProvider — theme + locale

```tsx
import { ConfigProvider } from 'antd';
import ko_KR from 'antd/locale/ko_KR';

<ConfigProvider
  locale={ko_KR}
  theme={{
    token: {
      colorPrimary: '#0064FF',
      borderRadius: 8,
      fontFamily: 'Pretendard',
    },
    components: {
      Button: { borderRadius: 4 },
    },
  }}
>
  <App />
</ConfigProvider>
```

## 4. 자주 쓰는 컴포넌트

### Button
```tsx
import { Button } from 'antd';

<Button type="primary">기본</Button>
<Button type="default">default</Button>
<Button type="dashed">dashed</Button>
<Button type="text">text</Button>
<Button type="link">link</Button>

<Button danger>위험</Button>
<Button loading>로딩</Button>
<Button icon={<SearchOutlined />}>검색</Button>
```

### Input / Form
```tsx
import { Form, Input, Button } from 'antd';

const onFinish = (values) => console.log(values);

<Form
  layout="vertical"
  onFinish={onFinish}
>
  <Form.Item
    label="이메일"
    name="email"
    rules={[
      { required: true, message: '필수' },
      { type: 'email', message: '이메일 형식' },
    ]}
  >
    <Input />
  </Form.Item>
  <Form.Item label="비밀번호" name="password" rules={[{ required: true }]}>
    <Input.Password />
  </Form.Item>
  <Button htmlType="submit" type="primary">로그인</Button>
</Form>
```

→ antd 의 Form 은 자체 form state + validation 가짐. **react-hook-form 과 양자택일**.

### Modal
```tsx
import { Modal } from 'antd';

<Modal
  open={open}
  onCancel={() => setOpen(false)}
  onOk={onConfirm}
  title="확인"
  okText="확인" cancelText="취소"
>
  본문
</Modal>

// 또는 명령형
Modal.confirm({
  title: '삭제하시겠습니까?',
  onOk: () => doDelete(),
});
```

### message / notification — toast
```tsx
import { message, notification } from 'antd';

message.success('저장 완료');
message.error('실패');
message.loading('업로드 중...', 2);

notification.success({
  message: '제목',
  description: '본문',
  placement: 'topRight',
});
```

→ v5 부터 `App` 컴포넌트 wrap 권장 (`<App><Routes/></App>`).

### Table
```tsx
import { Table } from 'antd';

const columns = [
  { title: '이름', dataIndex: 'name', key: 'name' },
  { title: '나이', dataIndex: 'age', key: 'age' },
  {
    title: '액션',
    key: 'action',
    render: (_, record) => <a onClick={() => edit(record)}>편집</a>,
  },
];

<Table
  columns={columns}
  dataSource={users}
  rowKey="id"
  pagination={{ pageSize: 10 }}
/>
```

→ B2B / admin 화면에 강력. sorting / filter / pagination 내장.

### DatePicker / Select / Upload
```tsx
import { DatePicker, Select, Upload, Button } from 'antd';

<DatePicker value={date} onChange={setDate} />
<DatePicker.RangePicker />

<Select
  value={value}
  onChange={setValue}
  options={[{ value: 'a', label: 'A' }, { value: 'b', label: 'B' }]}
/>

<Upload action="/upload">
  <Button>업로드</Button>
</Upload>
```

### Layout
```tsx
import { Layout } from 'antd';
const { Header, Sider, Content, Footer } = Layout;

<Layout style={{ minHeight: '100vh' }}>
  <Header>...</Header>
  <Layout>
    <Sider>...</Sider>
    <Content>...</Content>
  </Layout>
  <Footer>...</Footer>
</Layout>
```

## 5. icon

```tsx
import { SearchOutlined, DeleteOutlined } from '@ant-design/icons';

<SearchOutlined />
<DeleteOutlined style={{ fontSize: 18, color: 'red' }} />
```

→ tree-shaking 위해 named import.

## 6. v5 의 CSS-in-JS

antd v5 는 자동 CSS-in-JS. **dynamic theme + 자동 hash class**. v4 의 less 의 단점 (전체 CSS) 해소.

```tsx
// theme runtime 변경
const [dark, setDark] = useState(false);
<ConfigProvider theme={{ algorithm: dark ? theme.darkAlgorithm : theme.defaultAlgorithm }}>
```

## 7. v4 → v5 주요 변경

- icon 이 별도 패키지 `@ant-design/icons`.
- Form 의 API 큰 변화 (v3 → v4 에서 이미). v4 → v5 는 작은 변화.
- 디자인 토큰 (`theme.token.colorPrimary`).
- 일부 컴포넌트 deprecated (`Comment`, `PageHeader`).
- ConfigProvider 의 `prefixCls` 등 옵션.

## 8. Tailwind 와 공존 — job-answer-fe 패턴

```tsx
<Button
  type="primary"
  className="!bg-blue-600 !px-6"     // ! prefix = !important
>
  Click
</Button>
```

→ `!` 가 Tailwind 의 important variant. antd 의 자체 스타일 위로 덮어쓰기.

`tailwind.config.js`:
```ts
corePlugins: { preflight: false },   // antd reset 과 충돌 방지
```

## 9. job-answer-fe 패턴

```tsx
// App.tsx
<ConfigProvider locale={ko_KR} theme={{ token: { colorPrimary: '#...' } }}>
  <App>                              {/* message context */}
    <BrowserRouter>...</BrowserRouter>
  </App>
</ConfigProvider>

// 페이지
import { Form, Input, Button, message } from 'antd';

const Login = () => {
  const [api, contextHolder] = message.useMessage();

  const onFinish = async (values) => {
    try {
      await login(values);
      api.success('로그인 완료');
    } catch (e) {
      api.error('실패');
    }
  };

  return (
    <>
      {contextHolder}
      <Form onFinish={onFinish}>
        <Form.Item name="email" rules={[{ required: true, type: 'email' }]}>
          <Input placeholder="이메일" />
        </Form.Item>
        <Form.Item name="password" rules={[{ required: true }]}>
          <Input.Password placeholder="비밀번호" />
        </Form.Item>
        <Button htmlType="submit" type="primary" block>로그인</Button>
      </Form>
    </>
  );
};
```

## 10. antd Form vs react-hook-form

- **antd Form** — 내장 validation + UI 통합. antd 만 쓰면 자연.
- **react-hook-form** — 더 강력 / lib 독립 / zod 통합. antd Input 도 wrap 가능.

```tsx
// react-hook-form + antd
import { Controller } from 'react-hook-form';

<Controller
  control={control}
  name="email"
  render={({ field, fieldState }) => (
    <Form.Item validateStatus={fieldState.error ? 'error' : ''} help={fieldState.error?.message}>
      <Input {...field} />
    </Form.Item>
  )}
/>
```

→ 큰 form / zod 검증 원하면 react-hook-form.

## 11. 함정

1. **Modal.confirm 의 context 누락** — `App.useApp()` 또는 wrap 필요 (v5).
2. **Form 의 ref / initialValues** — controlled vs uncontrolled 혼선.
3. **bundle 크기** — 전체 import 가 무거움. icon 도 named import.
4. **CSS reset** — v5 의 reset.css 와 Tailwind preflight 충돌. preflight 끄거나 우선순위 관리.
5. **Table 의 rowKey** — `key` 가 unique 한지. 없으면 index 자동 + 경고.
6. **locale 누락** — DatePicker 영문. `ConfigProvider` 의 `locale={ko_KR}`.
7. **v4 사용 시 `babel-plugin-import`** — v5 는 불필요.

## 12. 외부 자료

- [Ant Design 공식](https://ant.design/)
- [antd 5.x changelog](https://ant.design/changelog)

## 13. 관련

- [[ui-libraries]]
- [[mui-material]]
- [[../styling/tailwind]]
- [[../forms/react-hook-form]]
