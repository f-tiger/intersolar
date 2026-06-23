# 多人实时协作 · Supabase 配置（5 分钟）

> 调研 App 默认是"纯本地"。按下面配置好 Supabase 后，App 里「数据/归档 → ☁️ 多人实时协作」即可启用：所有人**实时看同一份数据、照片自动入云、自动合并**。
> ⚠️ 现场 wifi 不稳时，照片上传可能失败——兜底仍可用「导出 JSON / 合并导入」。

## 1. 建项目
1. 去 https://supabase.com → 注册 → New project（免费档即可），选离慕尼黑近的区域（如 EU Frankfurt）。
2. 项目建好后，进 **Project Settings → API**，记下两样：
   - **Project URL**（形如 `https://xxxx.supabase.co`）
   - **anon public** key（`eyJ...`，这是**公开**密钥，可放前端）

## 2. 建表 + 权限（复制粘贴跑一次）
进 **SQL Editor → New query**，粘贴下面全部，点 **Run**：

```sql
-- 展位状态（每个展位一行，整体 last-write-wins）
create table if not exists public.booth_state (
  event text not null,
  booth_id text not null,
  data jsonb not null default '{}',
  updated_by text,
  updated_at timestamptz not null default now(),
  primary key (event, booth_id)
);

-- 照片元数据（文件本身存 Storage）
create table if not exists public.photos (
  id text primary key,
  event text not null,
  booth_id text not null,
  path text not null,
  caption text,
  by text,
  ts bigint
);

-- 行级安全：内部短期工具，允许 anon 读写（任何拿到 URL+key+活动代号的人可协作）
alter table public.booth_state enable row level security;
alter table public.photos enable row level security;
create policy "anon all booth_state" on public.booth_state for all to anon using (true) with check (true);
create policy "anon all photos"      on public.photos      for all to anon using (true) with check (true);

-- 开启实时
alter publication supabase_realtime add table public.booth_state;
alter publication supabase_realtime add table public.photos;
```

## 3. 建照片存储桶
1. 左侧 **Storage → New bucket** → 名字填 **`photos`** → 勾 **Public bucket** → 创建。
2. 再回 **SQL Editor** 跑这段（让协作者能上传/删除照片）：

```sql
create policy "anon upload photos" on storage.objects for insert to anon with check (bucket_id = 'photos');
create policy "anon read photos"   on storage.objects for select to anon using (bucket_id = 'photos');
create policy "anon delete photos" on storage.objects for delete to anon using (bucket_id = 'photos');
```

## 4. 在 App 里启用
1. 打开 https://intersolar-booth-scout.netlify.app → 底部 **数据/归档** → **☁️ 多人实时协作**。
2. 填：**Supabase URL**、**anon key**、**活动代号**（如 `intersolar2026`，大家必须一致）、**你的名字**。
3. 点 **启用 / 连接实时协作** → 页面刷新后顶部显示"✅ 实时协作中"。
4. 点 **复制分享链接给同事** → 把链接发群里。**同事点开即自动连同一份数据**，只需各自填名字。

## 5. 收工
- 数据在 Supabase，随时可在 App 里 **导出 HTML/JSON 归档**。
- 展会结束后，建议在 Supabase 删除该项目（或换活动代号），避免公开 key 长期暴露。

---

### 说明 / 限制
- **安全**：上面策略让"任何拿到 URL+anon key 的人"都能读写（适合内部 3 天用）。想更严可改用带密码登录或在策略里校验固定 `event` 值。
- **冲突**：同一展位以"最后保存者"为准（整条覆盖）——建议**按展位分工**避免两人同时改同一个。
- **照片**：压缩到长边≤1600px 后上传；失败会提示，可重拍或用导出兜底。
- **临时新增的展位**：仅本机可见，不会同步（请尽量用主清单里的展位）。
- 不配置 Supabase 时，App 仍是完整可用的离线本地版。
