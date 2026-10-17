# 5school — весь код проекта (версия с тёмной и светлой темой)

Файлы идут по порядку. Путь к каждому файлу указан в заголовке. Создайте такие же файлы в папке проекта и вставьте содержимое.

## `5school/.env.example`

```
NEXT_PUBLIC_SUPABASE_URL=https://your-project.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=your-anon-key
# В проде укажите адрес сайта, иначе ссылки в письмах поведут на localhost
NEXT_PUBLIC_SITE_URL=http://localhost:3000
```

## `5school/.gitignore`

```
node_modules/
.next/
.vercel/
.env*.local
next-env.d.ts
*.tsbuildinfo
.DS_Store
```

## `5school/DEPLOY.md`

````md
# Запуск 5school в интернете

Порядок важен. Если аккаунты уже есть, это 1–2 часа.

## 1. Что понадобится

- GitHub: репозиторий с проектом (файл `.env.local` в него попадать не должен, он в `.gitignore`).
- Supabase: **отдельный проект для продакшена**, не тот, где вы тестировали.
- Vercel: вход через GitHub.
- Resend (или любой другой SMTP) для писем.
- Домен.

## 2. База данных

В Supabase откройте SQL Editor и выполните файлы по очереди, каждый целиком:

1. `supabase/schema.sql`
2. `supabase/schema_stage3.sql`
3. `supabase/schema_stage4.sql`
4. `supabase/schema_stage5.sql`
5. `supabase/schema_stage6.sql`

Если файл выдал ошибку, остановитесь и пришлите мне её текст. Не пропускайте файл и не запускайте следующий.

Проверка, что защита строк включена на всех таблицах:

```sql
select tablename, rowsecurity from pg_tables where schemaname = 'public';
```

В колонке `rowsecurity` везде должно быть `true`.

Бакет `tutor-photos` (Storage) должен быть публичным, с лимитом 4 МБ и типами jpeg, png, webp. Лимит и типы задаёт `schema_stage5.sql`.

## 3. Авторизация

В Supabase, раздел Authentication, найдите настройки URL (URL Configuration):

- Site URL: `https://ВАШ-ДОМЕН`
- Redirect URLs: `https://ВАШ-ДОМЕН/auth/callback` и `http://localhost:3000/auth/callback`

Там же проверьте, что подтверждение почты включено. В настройках SMTP подключите свой SMTP (например, Resend): встроенная отправка Supabase рассчитана на тесты и быстро упирается в лимиты.

## 4. Vercel

1. Add New → Project → выберите репозиторий. Framework определится как Next.js.
2. В Environment Variables добавьте:
   - `NEXT_PUBLIC_SUPABASE_URL` и `NEXT_PUBLIC_SUPABASE_ANON_KEY` (Supabase → Project Settings → API)
   - `NEXT_PUBLIC_SITE_URL` = `https://ВАШ-ДОМЕН`
3. Ключ `service_role` не добавляйте никуда и не кладите в репозиторий. Приложению он не нужен.
4. Deploy. Затем Settings → Domains → добавьте домен и пропишите DNS-записи у регистратора.
5. После любой смены переменных нужен новый деплой (Redeploy).

## 5. Одобрение анкет

Новая анкета не видна в каталоге, пока вы её не одобрите. В Supabase → Table Editor → `tutor_profiles` поставьте `is_approved = true` у нужного репетитора или выполните:

```sql
update tutor_profiles set is_approved = true where id = 'ID-РЕПЕТИТОРА';
```

## 6. Проверка на боевом сайте

Нужны две разные почты: репетитор и ученик.

1. Регистрация репетитора, письмо, ссылка из письма: попадаете в кабинет.
2. Заполнить анкету, загрузить фото, включить «Показывать в каталоге». Видно сообщение «на проверке».
3. Одобрить анкету (шаг 5): она появилась в каталоге.
4. Репетитор добавляет слот на завтра и ещё один на ближайшие 5–10 минут.
5. Регистрация ученика, запись на слот: слот исчез, урок виден в кабинетах обоих.
6. Репетитор вставляет ссылку на урок: у ученика появилась кнопка «Войти на урок».
7. Ученик отменяет урок на завтра: слот вернулся в расписание.
8. Когда ближайший урок начался: «Урок прошёл», затем ученик оставляет отзыв, рейтинг в каталоге обновился.
9. Домашка: репетитор задаёт, ученик сдаёт, репетитор проверяет.
10. Выйти и открыть `/dashboard`: должен быть переход на `/login`.
11. Открыть ссылку из письма в другом браузере: понятное сообщение, а не пустая страница.

## 7. После запуска

- Ошибки: подключить Sentry (`npx @sentry/wizard@latest -i nextjs`). Можно сделать вместе.
- Статистика посещений: Vercel Analytics или Plausible.
- Резервные копии: проверьте, что входит в ваш тариф Supabase. Пока копий нет, хотя бы раз в неделю выгружайте базу.
- Страницы «Условия» и «Политика конфиденциальности» нужны до приёма первых настоящих пользователей. Для них нужны страна и данные юрлица. Черновик сделаю, проверить его должен юрист.
````

## `5school/app/actions.ts`

```ts
'use server';
import { randomUUID } from 'crypto';
import { redirect } from 'next/navigation';
import { supabaseServer } from '@/lib/supabase/server';
import { zonedToIso } from '@/lib/time';

const LOGIN_ERROR = 'Почта или пароль не подходят.';
const PHOTO_TYPES = ['image/jpeg', 'image/png', 'image/webp'];
const PHOTO_MAX = 4 * 1024 * 1024; // на Vercel тело запроса ограничено 4,5 МБ

export async function signUp(formData: FormData) {
  const supabase = await supabaseServer();
  const role = formData.get('role') === 'tutor' ? 'tutor' : 'student';
  const { data, error } = await supabase.auth.signUp({
    email: String(formData.get('email') ?? ''),
    password: String(formData.get('password') ?? ''),
    options: {
      data: { full_name: String(formData.get('name') ?? '').slice(0, 60), role },
      emailRedirectTo: `${process.env.NEXT_PUBLIC_SITE_URL}/auth/callback`,
    },
  });
  if (error) redirect(`/register?error=${encodeURIComponent('Не удалось создать аккаунт. Возможно, он уже есть.')}`);
  if (!data.session) redirect('/login?message=check-email');
  redirect('/dashboard');
}

export async function signIn(formData: FormData) {
  const supabase = await supabaseServer();
  const { error } = await supabase.auth.signInWithPassword({
    email: String(formData.get('email') ?? ''),
    password: String(formData.get('password') ?? ''),
  });
  if (error) {
    const msg = error.message.toLowerCase().includes('not confirmed') ? 'Подтвердите почту: ссылка уже в письме.' : LOGIN_ERROR;
    redirect(`/login?error=${encodeURIComponent(msg)}`);
  }
  redirect('/dashboard');
}

export async function signOut() {
  const supabase = await supabaseServer();
  await supabase.auth.signOut();
  redirect('/');
}

export async function saveTutorProfile(formData: FormData) {
  const supabase = await supabaseServer();
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) redirect('/login');
  const subjects = String(formData.get('subjects') ?? '').split(',').map((s) => s.trim()).filter(Boolean).slice(0, 10);
  const { error } = await supabase.from('tutor_profiles').update({
    headline: String(formData.get('headline') ?? '').slice(0, 120),
    bio: String(formData.get('bio') ?? '').slice(0, 2000),
    subjects,
    price_per_hour: Math.max(0, Math.round(Number(formData.get('price')) || 0)),
    is_published: formData.get('is_published') === 'on',
    updated_at: new Date().toISOString(),
  }).eq('id', user.id);
  if (error) redirect('/tutor/profile?error=1');
  redirect('/tutor/profile?saved=1');
}

export async function addPhoto(formData: FormData) {
  const supabase = await supabaseServer();
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) redirect('/login');
  const file = formData.get('photo');
  if (!(file instanceof File) || file.size === 0 || file.size > PHOTO_MAX || !PHOTO_TYPES.includes(file.type)) {
    redirect('/tutor/profile?error=photo');
  }
  const { data: t } = await supabase.from('tutor_profiles').select('photos').eq('id', user.id).single();
  const current: string[] = t?.photos ?? [];
  if (current.length >= 10) redirect('/tutor/profile?error=photo');
  const path = `${user.id}/${randomUUID()}`;
  const { error: upErr } = await supabase.storage.from('tutor-photos').upload(path, file, { contentType: file.type });
  if (upErr) redirect('/tutor/profile?error=photo');
  const { data: pub } = supabase.storage.from('tutor-photos').getPublicUrl(path);
  // Массив фото меняется только через функцию в базе: она проверяет, что ссылки ведут в папку этого репетитора
  const { error } = await supabase.rpc('set_my_photos', { p_photos: [...current, pub.publicUrl] });
  if (error) {
    await supabase.storage.from('tutor-photos').remove([path]);
    redirect('/tutor/profile?error=photo');
  }
  redirect('/tutor/profile?saved=1');
}

export async function removePhoto(formData: FormData) {
  const supabase = await supabaseServer();
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) redirect('/login');
  const url = String(formData.get('url') ?? '');
  const { data: t } = await supabase.from('tutor_profiles').select('photos').eq('id', user.id).single();
  const current: string[] = t?.photos ?? [];
  if (!current.includes(url)) redirect('/tutor/profile?error=1');
  const { error } = await supabase.rpc('set_my_photos', { p_photos: current.filter((p) => p !== url) });
  if (error) redirect('/tutor/profile?error=1');
  const path = url.split('/tutor-photos/')[1];
  if (path && path.startsWith(`${user.id}/`)) {
    await supabase.storage.from('tutor-photos').remove([path]);
  }
  redirect('/tutor/profile?saved=1');
}

export async function addSlot(formData: FormData) {
  const supabase = await supabaseServer();
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) redirect('/login');
  // Время вводится в поясе TZ (см. lib/time.ts) и переводится в UTC на сервере
  const startsAt = zonedToIso(String(formData.get('starts_at') ?? ''));
  if (!startsAt) redirect('/tutor/schedule?error=1');
  const { error } = await supabase.from('availability').insert({ tutor_id: user.id, starts_at: startsAt });
  redirect(error ? '/tutor/schedule?error=1' : '/tutor/schedule?saved=1');
}

export async function deleteSlot(formData: FormData) {
  const supabase = await supabaseServer();
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) redirect('/login');
  await supabase.from('availability').delete().eq('id', String(formData.get('id'))).eq('tutor_id', user.id);
  redirect('/tutor/schedule?saved=1');
}

export async function completeLesson(formData: FormData) {
  const supabase = await supabaseServer();
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) redirect('/login');
  // Функция в базе не даст завершить урок, который ещё не начался
  const { error } = await supabase.rpc('complete_lesson', { p_id: String(formData.get('id') ?? '') });
  redirect(error ? '/tutor/schedule?error=1' : '/tutor/schedule?saved=1');
}

export async function bookLesson(formData: FormData) {
  const supabase = await supabaseServer();
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) redirect('/login');
  const tutorId = String(formData.get('tutor_id') ?? '');
  const { error } = await supabase.rpc('book_lesson', { p_tutor: tutorId, p_starts: String(formData.get('starts_at') ?? '') });
  if (error) redirect(`/tutors/${tutorId}?error=${encodeURIComponent('Этот слот уже занят или запись недоступна.')}`);
  redirect('/dashboard?booked=1');
}

export async function saveLessonPlan(formData: FormData) {
  const supabase = await supabaseServer();
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) redirect('/login');
  await supabase.from('lessons')
    .update({ plan: String(formData.get('plan') ?? '').slice(0, 4000) })
    .eq('id', String(formData.get('id'))).eq('tutor_id', user.id);
  redirect('/tutor/schedule?saved=1');
}

export async function addHomework(formData: FormData) {
  const supabase = await supabaseServer();
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) redirect('/login');
  const { error } = await supabase.from('homework').insert({
    lesson_id: String(formData.get('lesson_id') ?? ''),
    title: String(formData.get('title') ?? '').trim().slice(0, 120),
    body: String(formData.get('body') ?? '').slice(0, 4000),
  });
  redirect(error ? '/tutor/schedule?error=1' : '/tutor/schedule?saved=1');
}

export async function deleteHomework(formData: FormData) {
  const supabase = await supabaseServer();
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) redirect('/login');
  await supabase.from('homework').delete().eq('id', String(formData.get('id')));
  redirect('/tutor/schedule?saved=1');
}

export async function submitHomework(formData: FormData) {
  const supabase = await supabaseServer();
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) redirect('/login');
  const { error } = await supabase.rpc('submit_homework', {
    p_id: String(formData.get('id') ?? ''),
    p_answer: String(formData.get('answer') ?? '').slice(0, 4000),
  });
  redirect(error ? '/dashboard?error=1' : '/dashboard?saved=1');
}

export async function checkHomework(formData: FormData) {
  const supabase = await supabaseServer();
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) redirect('/login');
  const { error } = await supabase.rpc('check_homework', {
    p_id: String(formData.get('id') ?? ''),
    p_feedback: String(formData.get('feedback') ?? '').slice(0, 4000),
  });
  redirect(error ? '/tutor/schedule?error=1' : '/tutor/schedule?saved=1');
}

// Отзыв: только после завершённого урока (проверяет база). Повторный отзыв обновляет старый.
export async function saveReview(formData: FormData) {
  const supabase = await supabaseServer();
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) redirect('/login');
  const rating = Math.min(5, Math.max(1, Math.round(Number(formData.get('rating')) || 0)));
  const { error } = await supabase.from('reviews').upsert(
    {
      tutor_id: String(formData.get('tutor_id') ?? ''),
      student_id: user.id,
      rating,
      body: String(formData.get('body') ?? '').slice(0, 1000),
    },
    { onConflict: 'tutor_id,student_id' },
  );
  redirect(error ? '/dashboard?error=1' : '/dashboard?saved=1');
}

// Ссылка на видеозвонок: только https, иначе в href можно подсунуть javascript:
export async function saveLessonLink(formData: FormData) {
  const supabase = await supabaseServer();
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) redirect('/login');
  const raw = String(formData.get('meeting_url') ?? '').trim();
  let url = '';
  if (raw) {
    try {
      const u = new URL(raw);
      if (u.protocol === 'https:') url = u.toString();
    } catch {
      // неверный адрес: ниже вернём ошибку
    }
    if (!url || url.length > 300) redirect('/tutor/schedule?error=1');
  }
  const { error } = await supabase.from('lessons').update({ meeting_url: url })
    .eq('id', String(formData.get('id') ?? '')).eq('tutor_id', user.id);
  redirect(error ? '/tutor/schedule?error=1' : '/tutor/schedule?saved=1');
}

// Отмена урока до начала. Кто отменяет и можно ли, решает база (cancel_lesson).
export async function cancelLesson(formData: FormData) {
  const supabase = await supabaseServer();
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) redirect('/login');
  const back = formData.get('from') === 'tutor' ? '/tutor/schedule' : '/dashboard';
  const { error } = await supabase.rpc('cancel_lesson', { p_id: String(formData.get('id') ?? '') });
  redirect(error ? `${back}?error=1` : `${back}?saved=1`);
}
```

## `5school/app/auth/callback/route.ts`

```ts
import { NextResponse, type NextRequest } from 'next/server';
import { supabaseServer } from '@/lib/supabase/server';

// Ссылка из письма подтверждения почты
export async function GET(req: NextRequest) {
  const code = req.nextUrl.searchParams.get('code');
  if (!code) return NextResponse.redirect(new URL('/login', req.url));
  const supabase = await supabaseServer();
  const { error } = await supabase.auth.exchangeCodeForSession(code);
  if (error) {
    // Частый случай: ссылку открыли в другом браузере, чем тот, где регистрировались
    const msg = 'Ссылка не сработала. Если вы открыли её в другом браузере, почта, скорее всего, уже подтверждена: попробуйте войти по паролю.';
    return NextResponse.redirect(new URL(`/login?error=${encodeURIComponent(msg)}`, req.url));
  }
  return NextResponse.redirect(new URL('/dashboard', req.url));
}
```

## `5school/app/components/cancel-lesson.tsx`

```tsx
import { cancelLesson } from '@/app/actions';

// Отмена в два шага без JavaScript: сначала раскрывается подтверждение
export function CancelLesson({ id, from }: { id: string; from: 'student' | 'tutor' }) {
  return (
    <details className="cancel">
      <summary>Отменить</summary>
      <form action={cancelLesson}>
        <input type="hidden" name="id" value={id} />
        <input type="hidden" name="from" value={from} />
        <button className="ghost">Да, отменить урок</button>
      </form>
    </details>
  );
}
```

## `5school/app/components/theme-toggle.tsx`

```tsx
'use client';

// Тёмная тема по умолчанию. Выбор запоминается в cookie, и сервер сразу отдаёт страницу в нужной теме (без вспышки).
export function ThemeToggle() {
  function toggle() {
    const root = document.documentElement;
    const next = root.dataset.theme === 'light' ? 'dark' : 'light';
    root.dataset.theme = next;
    document.cookie = `theme=${next}; path=/; max-age=31536000; samesite=lax`;
  }
  return (
    <button type="button" className="theme" onClick={toggle} aria-label="Сменить тему" title="Сменить тему">
      <svg className="i-sun" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" aria-hidden="true">
        <circle cx="12" cy="12" r="4" />
        <path d="M12 2v2M12 20v2M4.9 4.9l1.4 1.4M17.7 17.7l1.4 1.4M2 12h2M20 12h2M4.9 19.1l1.4-1.4M17.7 6.3l1.4-1.4" />
      </svg>
      <svg className="i-moon" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round" aria-hidden="true">
        <path d="M21 12.8A9 9 0 1 1 11.2 3a7 7 0 0 0 9.8 9.8z" />
      </svg>
    </button>
  );
}
```

## `5school/app/dashboard/page.tsx`

```tsx
import { redirect } from 'next/navigation';
import { supabaseServer } from '@/lib/supabase/server';
import { signOut, submitHomework, saveReview } from '@/app/actions';
import { fmt } from '@/lib/time';
import { CancelLesson } from '@/app/components/cancel-lesson';

const STATUS: Record<string, string> = { scheduled: 'запланирован', completed: 'проведён', cancelled: 'отменён' };

export default async function Dashboard({ searchParams }: { searchParams: Promise<{ booked?: string; saved?: string; error?: string }> }) {
  const sp = await searchParams;
  const supabase = await supabaseServer();
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) redirect('/login');
  const { data: p } = await supabase.from('profiles').select('full_name, role').eq('id', user.id).single();
  const isTutor = p?.role === 'tutor';

  const { data: lessons } = await supabase
    .from('lessons')
    .select('id, tutor_id, starts_at, status, plan, meeting_url, profiles(full_name), tutor_profiles(profiles(full_name))')
    .or(`student_id.eq.${user.id},tutor_id.eq.${user.id}`)
    .order('starts_at');
  const list = (lessons ?? []) as any[];
  const now = new Date();

  const { data: hw } = isTutor
    ? { data: [] as any[] }
    : await supabase.from('homework').select('id, title, body, status, feedback').order('created_at');
  const { data: myReviews } = isTutor
    ? { data: [] as any[] }
    : await supabase.from('reviews').select('tutor_id, rating, body').eq('student_id', user.id);
  const reviewMap = new Map<string, any>((myReviews ?? []).map((r: any): [string, any] => [r.tutor_id, r]));

  // Репетиторы, у которых был завершённый урок: им можно оставить отзыв
  const toReview = new Map<string, string>();
  list.filter((l) => l.status === 'completed').forEach((l) => {
    toReview.set(l.tutor_id, l.tutor_profiles?.profiles?.full_name ?? 'Репетитор');
  });

  return (
    <main className="wrap">
      <h1>Здравствуйте, {p?.full_name}</h1>
      <p className="muted">{isTutor ? 'Кабинет репетитора' : 'Кабинет ученика'}</p>
      {sp.booked && <p className="notice">Урок записан.</p>}
      {sp.saved && <p className="notice">Сохранено.</p>}
      {sp.error && <p className="err">Не получилось сохранить. Попробуйте ещё раз.</p>}

      <div className="grid">
        {isTutor ? (
          <>
            <a className="card" href="/tutor/profile"><h3>Моя анкета</h3><p className="muted">Предметы, цена, фото и описание.</p></a>
            <a className="card" href="/tutor/schedule"><h3>Расписание</h3><p className="muted">Уроки, планы и домашка.</p></a>
          </>
        ) : (
          <a className="card" href="/"><h3>Найти репетитора</h3><p className="muted">Каталог с рейтингом и отзывами.</p></a>
        )}
      </div>

      <section className="card">
        <h2>Мои уроки</h2>
        {list.length === 0 && <p className="muted">Уроков пока нет.</p>}
        <ul className="list">
          {list.map((l) => (
            <li key={l.id}>
              <span>{fmt(l.starts_at)}</span>
              <span className="muted">
                {isTutor ? l.profiles?.full_name : l.tutor_profiles?.profiles?.full_name} · {STATUS[l.status] ?? l.status}
              </span>
              {l.status === 'scheduled' && (
                <span className="row">
                  {l.meeting_url && <a href={l.meeting_url} target="_blank" rel="noopener noreferrer">Войти на урок</a>}
                  {new Date(l.starts_at) > now && <CancelLesson id={l.id} from={isTutor ? 'tutor' : 'student'} />}
                </span>
              )}
            </li>
          ))}
        </ul>
      </section>

      {!isTutor && list.some((l) => l.plan) && (
        <section className="card">
          <h2>План уроков</h2>
          {list.filter((l) => l.plan).map((l) => (
            <div className="hw" key={l.id}>
              <p><strong>{fmt(l.starts_at)}</strong></p>
              <p>{l.plan}</p>
            </div>
          ))}
        </section>
      )}

      {!isTutor && (
        <section className="card">
          <h2>Домашка</h2>
          {(hw ?? []).length === 0 && <p className="muted">Домашних заданий пока нет.</p>}
          {(hw ?? []).map((h: any) => (
            <div className="hw" key={h.id}>
              <p><strong>{h.title}</strong></p>
              {h.body && <p className="muted">{h.body}</p>}
              {h.status === 'assigned' && (
                <form action={submitHomework}>
                  <input type="hidden" name="id" value={h.id} />
                  <label>Ваш ответ<textarea name="answer" rows={3} required maxLength={4000} /></label>
                  <button className="primary">Отправить</button>
                </form>
              )}
              {h.status === 'submitted' && <p className="muted">Отправлено, ждёт проверки.</p>}
              {h.status === 'checked' && <p>Проверено.{h.feedback && ` Комментарий: ${h.feedback}`}</p>}
            </div>
          ))}
        </section>
      )}

      {!isTutor && (
        <section className="card">
          <h2>Отзывы</h2>
          {toReview.size === 0 && <p className="muted">Оставить отзыв можно после завершённого урока.</p>}
          {[...toReview.entries()].map(([tutorId, name]) => {
            const mine = reviewMap.get(tutorId);
            return (
              <form action={saveReview} className="hw" key={tutorId}>
                <input type="hidden" name="tutor_id" value={tutorId} />
                <p><strong>{name}</strong>{mine && <span className="muted"> · отзыв сохранён</span>}</p>
                <label>Оценка
                  <select name="rating" defaultValue={mine?.rating ?? 5}>
                    {[5, 4, 3, 2, 1].map((n) => <option key={n} value={n}>{n}</option>)}
                  </select>
                </label>
                <label>Комментарий<textarea name="body" rows={2} maxLength={1000} defaultValue={mine?.body ?? ''} /></label>
                <button className="primary">Сохранить отзыв</button>
              </form>
            );
          })}
        </section>
      )}

      <form action={signOut}><button className="ghost">Выйти</button></form>
    </main>
  );
}
```

## `5school/app/globals.css`

```css
:root{color-scheme:dark;--paper:#0A0D14;--ink:#F3F5F8;--muted:#98A1B3;--line:rgba(255,255,255,.12);--line-strong:rgba(255,255,255,.3);--panel:rgba(255,255,255,.05);--field:rgba(255,255,255,.06);--accent:#2DD4BF;--soft:rgba(45,212,191,.14);--danger:#F87171;--star:#FBBF24;--glass:rgba(14,18,27,.62);--card-a:rgba(255,255,255,.07);--card-b:rgba(255,255,255,.03);--shadow:none;--sub:rgba(255,255,255,.04);--bar:rgba(255,255,255,.06);--bar-field:rgba(10,13,20,.6);--option-bg:#141824;--glow:rgba(45,212,191,.18)}
:root[data-theme=light]{color-scheme:light;--paper:#F5F4EF;--ink:#1E2433;--muted:#5F6879;--line:rgba(30,36,51,.14);--line-strong:rgba(30,36,51,.38);--panel:rgba(255,255,255,.7);--field:#fff;--accent:#0F766E;--soft:rgba(15,118,110,.1);--danger:#B42318;--star:#D99A00;--glass:rgba(255,255,255,.72);--card-a:rgba(255,255,255,.92);--card-b:rgba(255,255,255,.7);--shadow:0 1px 2px rgba(30,36,51,.05),0 10px 30px rgba(30,36,51,.06);--sub:rgba(30,36,51,.04);--bar:rgba(255,255,255,.6);--bar-field:rgba(255,255,255,.9);--option-bg:#fff;--glow:rgba(15,118,110,.12)}
*{box-sizing:border-box}
body{margin:0;background:radial-gradient(ellipse 80% 100% at 50% 0%,var(--glow),transparent 70%) top/100% 640px no-repeat,var(--paper);color:var(--ink);font:400 16px/1.5 "Golos Text",system-ui,sans-serif;min-height:100vh}
h1,h2,h3,.brand{font-family:"Golos Text",system-ui,sans-serif;font-weight:600;line-height:1.15;letter-spacing:-.02em;margin:0 0 12px}
h1{font-size:clamp(1.9rem,4.5vw,2.6rem)}
h2{font-size:1.4rem}
h3{font-size:1.1rem}
a{color:inherit}
/* Плавающая шапка-«таблетка» */
.top{position:sticky;top:12px;z-index:50;display:flex;justify-content:space-between;align-items:center;gap:12px;width:calc(100% - 32px);max-width:720px;margin:12px auto 0;padding:8px 8px 8px 20px;border:1px solid var(--line);border-radius:999px;background:var(--glass);box-shadow:var(--shadow);-webkit-backdrop-filter:blur(16px);backdrop-filter:blur(16px)}
.brand{font-size:18px;text-decoration:none;margin:0}
.brand::before{content:"";display:inline-block;width:10px;height:10px;background:var(--accent);border-radius:50%;margin-right:8px}
.nav{display:flex;align-items:center;gap:4px}
.nav a{padding:8px 14px;border-radius:999px;text-decoration:none;color:var(--muted);font-size:15px;white-space:nowrap}
.nav a:hover{color:var(--ink)}
.nav a.pill{background:var(--ink);color:var(--paper);font-weight:600}
.nav a.pill:hover{color:var(--paper);opacity:.9}
@media(max-width:520px){.nav .hide-sm{display:none}.top{padding-left:16px}}
@media(max-width:380px){.nav a{padding:8px 10px;font-size:14px}}
.wrap{max-width:1040px;margin:0 auto;padding:40px 24px 88px}
.narrow{max-width:520px}
/* Карточки «матовое стекло» */
.card{display:block;background:linear-gradient(180deg,var(--card-a),var(--card-b));box-shadow:var(--shadow);border:1px solid var(--line);border-radius:20px;padding:20px;text-decoration:none;margin-bottom:14px;-webkit-backdrop-filter:blur(14px);backdrop-filter:blur(14px)}
a.card:hover{border-color:var(--line-strong)}
.grid{display:grid;grid-template-columns:repeat(auto-fill,minmax(240px,1fr));gap:14px}
.tutor img,.tutor .ph{width:100%;aspect-ratio:4/3;object-fit:cover;border-radius:14px;background:linear-gradient(135deg,rgba(45,212,191,.28),rgba(99,102,241,.22));margin-bottom:12px}
.tags{display:flex;flex-wrap:wrap;gap:6px;margin:8px 0}
.tags span{background:var(--soft);color:var(--accent);padding:2px 10px;border-radius:99px;font-size:13px}
.stars{color:var(--star);letter-spacing:1px}
.price{font-weight:600}
.muted{color:var(--muted)}
.hero{display:flex;justify-content:space-between;gap:24px;flex-wrap:wrap}
.buy{text-align:right}
.photos{display:grid;grid-template-columns:repeat(auto-fill,minmax(160px,1fr));gap:10px;margin-bottom:14px}
.photos img{width:100%;aspect-ratio:1;object-fit:cover;border-radius:14px}
label{display:block;font-size:14px;font-weight:500;margin-bottom:14px}
input:not([type=radio]):not([type=checkbox]),textarea{display:block;width:100%;margin-top:6px;padding:11px 14px;border:1px solid var(--line);border-radius:12px;background:var(--field);color:var(--ink);font:inherit;font-size:16px}
input:focus-visible,textarea:focus-visible,select:focus-visible{outline:2px solid var(--accent);outline-offset:1px}
::placeholder{color:var(--muted);opacity:.85}
input[type=radio],input[type=checkbox]{accent-color:var(--accent)}
option{background:var(--option-bg);color:var(--ink)}
.check{display:flex;gap:8px;align-items:center}
.roles{display:grid;grid-template-columns:1fr 1fr;gap:10px;margin-bottom:14px}
.role{margin:0;position:relative}
.role input{position:absolute;opacity:0}
.role span{display:block;padding:12px;border:1px solid var(--line);border-radius:14px;cursor:pointer}
.role small{display:block;color:var(--muted);font-size:13px}
.role input:checked+span{border-color:var(--accent);background:var(--soft)}
.primary,.ghost{font:inherit;padding:12px 20px;border-radius:999px;cursor:pointer}
.primary{width:100%;background:var(--ink);color:var(--paper);border:0;font-weight:600}
.primary:hover{opacity:.92}
.primary:disabled{opacity:.5;cursor:default}
.ghost{background:none;border:1px solid var(--line);color:var(--ink)}
.ghost:hover{border-color:var(--line-strong)}
.err{color:var(--danger)}
.notice{color:var(--accent)}
.review p{margin:6px 0}
.slots{display:flex;flex-wrap:wrap;gap:8px;margin-bottom:12px}
.slots form,.list form{margin:0}
.list{list-style:none;padding:0;margin:0}
.list li{display:flex;justify-content:space-between;align-items:center;gap:12px;padding:10px 0;border-bottom:1px solid var(--line)}
.slot{display:flex;gap:8px;align-items:center;margin-bottom:14px;flex-wrap:wrap}
.slot input[type=datetime-local]{width:auto;margin-top:0}
.slot input[type=file]{margin-top:0}
.photos form{margin:0}
.photos form button{margin-top:6px;width:100%}
.list li.lesson{display:block}
.row{display:flex;justify-content:space-between;align-items:center;gap:12px;flex-wrap:wrap}
.hw{background:var(--sub);border:1px solid var(--line);border-radius:14px;padding:12px;margin:10px 0}
.hw p{margin:4px 0}
.hw form{margin:8px 0 0}
select{display:block;margin-top:6px;padding:10px 14px;border:1px solid var(--line);border-radius:12px;background:var(--field);color:var(--ink);font:inherit;font-size:16px}
.cancel summary{cursor:pointer;color:var(--muted);font-size:14px}
.cancel form{margin:8px 0 0}
/* Главная: огромный заголовок, под него наезжает матовая панель поиска */
.home-hero{text-align:center;padding:56px 0 28px}
.chip{display:inline-block;margin:0 0 20px;padding:6px 14px;border:1px solid var(--line);border-radius:999px;background:var(--panel);color:var(--muted);font-size:14px}
.home-title{position:relative;z-index:0;font-size:clamp(2.4rem,13vw,9rem);font-weight:600;letter-spacing:-.045em;line-height:.9;margin:0 0 -.28em}
.filters{position:relative;z-index:1;display:grid;grid-template-columns:2fr 1fr 1fr auto;gap:10px;align-items:center;max-width:860px;margin:0 auto 20px;padding:12px;text-align:left;border:1px solid var(--line);border-radius:28px;background:var(--bar);-webkit-backdrop-filter:blur(22px);backdrop-filter:blur(22px)}
.filters input,.filters select{margin-top:0;width:100%;background:var(--bar-field)}
.filters .primary{width:auto}
@media(max-width:640px){.filters{grid-template-columns:1fr;border-radius:22px}.filters .primary{width:100%}.home-title{margin-bottom:-.1em}}
.hero-cta{margin:0}
.cta{display:inline-block;padding:13px 28px;border:1px solid var(--ink);border-radius:999px;text-decoration:none;font-weight:500}
.cta:hover{background:var(--ink);color:var(--paper)}
/* Переключатель темы: плавающая кнопка справа внизу. В тёмной теме солнце, в светлой луна */
.theme{position:fixed;right:20px;bottom:20px;z-index:60;display:inline-flex;align-items:center;justify-content:center;width:44px;height:44px;padding:0;border:1px solid var(--line);border-radius:50%;background:var(--glass);box-shadow:var(--shadow);-webkit-backdrop-filter:blur(12px);backdrop-filter:blur(12px);color:var(--ink);cursor:pointer}
.theme:hover{border-color:var(--line-strong)}
.theme svg{display:block;width:18px;height:18px}
.theme .i-moon{display:none}
:root[data-theme=light] .theme .i-sun{display:none}
:root[data-theme=light] .theme .i-moon{display:block}
.list li>.muted{flex:1}
.slot .primary{width:auto}
.list li.lesson>form{margin-top:14px}
```

## `5school/app/layout.tsx`

```tsx
import type { Metadata } from 'next';
import { cookies } from 'next/headers';
import { hasSessionCookie } from '@/lib/session';
import { ThemeToggle } from '@/app/components/theme-toggle';
import './globals.css';

export const metadata: Metadata = { title: '5school — репетиторы', description: 'Занятия с репетиторами: отзывы, цены, расписание' };

export default async function RootLayout({ children }: { children: React.ReactNode }) {
  const signedIn = await hasSessionCookie();
  const theme = (await cookies()).get('theme')?.value === 'light' ? 'light' : 'dark';
  return (
    <html lang="ru" data-theme={theme}>
      <head>
        <link rel="preconnect" href="https://fonts.gstatic.com" crossOrigin="" />
        <link href="https://fonts.googleapis.com/css2?family=Golos+Text:wght@400;500;600;700&display=swap" rel="stylesheet" />
      </head>
      <body>
        <header className="top">
          <a className="brand" href="/">5school</a>
          <nav className="nav">
            <a className="hide-sm" href="/">Репетиторы</a>
            {signedIn ? (
              <a className="pill" href="/dashboard">Кабинет</a>
            ) : (
              <>
                <a href="/login">Войти</a>
                <a className="pill" href="/register">Регистрация</a>
              </>
            )}
          </nav>
        </header>
        {children}
        <ThemeToggle />
      </body>
    </html>
  );
}
```

## `5school/app/login/page.tsx`

```tsx
import { signIn } from '@/app/actions';

export default async function LoginPage({ searchParams }: { searchParams: Promise<{ error?: string; message?: string }> }) {
  const sp = await searchParams;
  return (
    <main className="wrap narrow">
      <h1>Вход в 5school</h1>
      {sp.message === 'check-email' && <p className="notice">Письмо отправлено. Подтвердите почту и войдите.</p>}
      {sp.error && <p className="err">{sp.error}</p>}
      <form action={signIn} className="card">
        <label>Почта<input name="email" type="email" required /></label>
        <label>Пароль<input name="password" type="password" required /></label>
        <button className="primary">Войти</button>
      </form>
      <p className="muted">Нет аккаунта? <a href="/register">Регистрация</a></p>
    </main>
  );
}
```

## `5school/app/page.tsx`

```tsx
import { supabaseServer } from '@/lib/supabase/server';
import { hasSessionCookie } from '@/lib/session';

type Row = { id: string; headline: string; subjects: string[]; price_per_hour: number; currency: string; rating_avg: number; rating_count: number; photos: string[]; profiles: { full_name: string } | null };
const stars = (v: number) => '★'.repeat(Math.round(v)) + '☆'.repeat(5 - Math.round(v));

export default async function Home({ searchParams }: { searchParams: Promise<{ q?: string; max?: string; sort?: string }> }) {
  const sp = await searchParams;
  const signedIn = await hasSessionCookie();
  const supabase = await supabaseServer();
  const { data } = await supabase
    .from('tutor_profiles')
    .select('id, headline, subjects, price_per_hour, currency, rating_avg, rating_count, photos, profiles(full_name)')
    .eq('is_published', true)
    .eq('is_approved', true);
  const all = (data ?? []) as unknown as Row[];

  // Фильтры в коде, а не в запросе: предметы вводятся свободным текстом, регистр и написание разные
  const q = (sp.q ?? '').trim().toLowerCase();
  const max = Number(sp.max);
  const sort = sp.sort === 'price' ? 'price' : 'rating';
  let tutors = all;
  if (q) tutors = tutors.filter((t) => [t.profiles?.full_name ?? '', t.headline, ...t.subjects].some((s) => s.toLowerCase().includes(q)));
  if (max > 0) tutors = tutors.filter((t) => t.price_per_hour <= max);
  tutors = [...tutors].sort(
    sort === 'price'
      ? (a, b) => a.price_per_hour - b.price_per_hour
      : (a, b) => b.rating_avg - a.rating_avg || b.rating_count - a.rating_count,
  );

  return (
    <main className="wrap">
      <section className="home-hero">
        <p className="chip">Живые уроки, расписание и домашка в одном месте</p>
        <h1 className="home-title">Репетиторы</h1>
        <form className="filters" method="get">
          <input name="q" placeholder="Предмет или имя" defaultValue={sp.q ?? ''} />
          <input name="max" type="number" min={0} placeholder="Цена до, в час" defaultValue={sp.max ?? ''} />
          <select name="sort" defaultValue={sort}>
            <option value="rating">По рейтингу</option>
            <option value="price">Сначала дешёвые</option>
          </select>
          <button className="primary">Найти</button>
        </form>
        {!signedIn && <p className="hero-cta"><a className="cta" href="/register">Стать репетитором</a></p>}
      </section>
      {all.length === 0 && <p className="muted">Пока нет опубликованных анкет.</p>}
      {all.length > 0 && tutors.length === 0 && <p className="muted">Ничего не найдено. Попробуйте изменить запрос.</p>}
      <div className="grid">
        {tutors.map((t) => (
          <a className="card tutor" href={`/tutors/${t.id}`} key={t.id}>
            {t.photos[0] ? <img src={t.photos[0]} alt="" /> : <div className="ph" />}
            <h3>{t.profiles?.full_name}</h3>
            <p className="muted">{t.headline}</p>
            <div className="tags">{t.subjects.map((s) => <span key={s}>{s}</span>)}</div>
            <p><span className="stars">{stars(t.rating_avg)}</span> <span className="muted">{t.rating_count} отзывов</span></p>
            <p className="price">{t.price_per_hour} {t.currency} / час</p>
          </a>
        ))}
      </div>
    </main>
  );
}
```

## `5school/app/register/page.tsx`

```tsx
import { signUp } from '@/app/actions';

export default async function RegisterPage({ searchParams }: { searchParams: Promise<{ error?: string }> }) {
  const sp = await searchParams;
  return (
    <main className="wrap narrow">
      <h1>Регистрация</h1>
      {sp.error && <p className="err">{sp.error}</p>}
      <form action={signUp} className="card">
        <label>Имя<input name="name" required maxLength={60} /></label>
        <div className="roles">
          <label className="role"><input type="radio" name="role" value="student" defaultChecked /><span><b>Ученик</b><small>Ищу репетитора</small></span></label>
          <label className="role"><input type="radio" name="role" value="tutor" /><span><b>Репетитор</b><small>Веду уроки</small></span></label>
        </div>
        <label>Почта<input name="email" type="email" required /></label>
        <label>Пароль<input name="password" type="password" minLength={6} required /></label>
        <button className="primary">Создать аккаунт</button>
      </form>
      <p className="muted">Уже есть аккаунт? <a href="/login">Войти</a></p>
    </main>
  );
}
```

## `5school/app/tutor/profile/page.tsx`

```tsx
import { redirect } from 'next/navigation';
import { supabaseServer } from '@/lib/supabase/server';
import { saveTutorProfile, addPhoto, removePhoto } from '@/app/actions';

export default async function TutorProfilePage({ searchParams }: { searchParams: Promise<{ saved?: string; error?: string }> }) {
  const sp = await searchParams;
  const supabase = await supabaseServer();
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) redirect('/login');
  const { data: t } = await supabase.from('tutor_profiles').select('*').eq('id', user.id).maybeSingle();
  if (!t) redirect('/dashboard');

  return (
    <main className="wrap narrow">
      <h1>Моя анкета</h1>
      {t.is_published && !t.is_approved && <p className="muted">Анкета на проверке: в каталоге она появится после одобрения.</p>}
      {sp.saved && <p className="notice">Сохранено.</p>}
      {sp.error && <p className="err">{sp.error === 'photo' ? 'Не удалось загрузить фото. Нужен JPEG, PNG или WebP до 4 МБ (всего не больше 10 фото).' : 'Не удалось сохранить. Попробуйте ещё раз.'}</p>}

      <form action={saveTutorProfile} className="card">
        <label>Заголовок<input name="headline" maxLength={120} defaultValue={t.headline} /></label>
        <label>О себе<textarea name="bio" rows={6} maxLength={2000} defaultValue={t.bio} /></label>
        <label>Предметы (через запятую)<input name="subjects" defaultValue={t.subjects.join(', ')} /></label>
        <label>Цена за час, {t.currency}<input name="price" type="number" min={0} defaultValue={t.price_per_hour} /></label>
        <label className="check"><input type="checkbox" name="is_published" defaultChecked={t.is_published} /> Показывать в каталоге</label>
        <button className="primary">Сохранить</button>
      </form>

      <section className="card">
        <h2>Фото</h2>
        <div className="photos">
          {t.photos.map((p: string) => (
            <form action={removePhoto} key={p}>
              <img src={p} alt="" />
              <input type="hidden" name="url" value={p} />
              <button className="ghost">Удалить</button>
            </form>
          ))}
        </div>
        {t.photos.length < 10 && (
          <form action={addPhoto} className="slot">
            <input type="file" name="photo" accept="image/jpeg,image/png,image/webp" required />
            <button className="primary">Загрузить</button>
          </form>
        )}
      </section>
    </main>
  );
}
```

## `5school/app/tutor/schedule/page.tsx`

```tsx
import { redirect } from 'next/navigation';
import { supabaseServer } from '@/lib/supabase/server';
import { addSlot, deleteSlot, completeLesson, saveLessonPlan, saveLessonLink, addHomework, deleteHomework, checkHomework } from '@/app/actions';
import { CancelLesson } from '@/app/components/cancel-lesson';
import { fmt, TZ } from '@/lib/time';

const HW: Record<string, string> = { assigned: 'не сдано', submitted: 'ждёт проверки', checked: 'проверено' };

export default async function SchedulePage({ searchParams }: { searchParams: Promise<{ saved?: string; error?: string }> }) {
  const sp = await searchParams;
  const supabase = await supabaseServer();
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) redirect('/login');
  const now = new Date();
  const { data: slots } = await supabase.from('availability').select('id, starts_at').eq('tutor_id', user.id).order('starts_at');
  const { data: lessons } = await supabase.from('lessons').select('id, starts_at, status, plan, meeting_url, profiles(full_name)').eq('tutor_id', user.id).order('starts_at');
  const ids = (lessons ?? []).map((l: any) => l.id);
  const { data: hw } = ids.length
    ? await supabase.from('homework').select('id, lesson_id, title, body, status, answer, feedback').in('lesson_id', ids).order('created_at')
    : { data: [] as any[] };

  return (
    <main className="wrap narrow">
      <h1>Расписание</h1>
      {sp.saved && <p className="notice">Сохранено.</p>}
      {sp.error && <p className="err">Что-то пошло не так. Проверьте данные и попробуйте снова.</p>}

      <section className="card">
        <h2>Свободное время</h2>
        <form action={addSlot} className="slot">
          <input type="datetime-local" name="starts_at" required />
          <button className="primary">Добавить</button>
        </form>
        <p className="muted">Время указывается по поясу {TZ}.</p>
        {(slots ?? []).length === 0 && <p className="muted">Слотов пока нет.</p>}
        <ul className="list">
          {(slots ?? []).map((s: any) => (
            <li key={s.id}>
              <span>{fmt(s.starts_at)}</span>
              <form action={deleteSlot}><input type="hidden" name="id" value={s.id} /><button className="ghost">Убрать</button></form>
            </li>
          ))}
        </ul>
      </section>

      <section className="card">
        <h2>Уроки</h2>
        {(lessons ?? []).length === 0 && <p className="muted">Записей пока нет.</p>}
        <ul className="list">
          {(lessons ?? []).map((l: any) => (
            <li className="lesson" key={l.id}>
              <div className="row">
                <span>{fmt(l.starts_at)} · {l.profiles?.full_name}</span>
                {l.status === 'scheduled'
                  ? (new Date(l.starts_at) <= now
                    ? <form action={completeLesson}><input type="hidden" name="id" value={l.id} /><button className="ghost">Урок прошёл</button></form>
                    : <span className="row"><span className="muted">Ещё не начался</span><CancelLesson id={l.id} from="tutor" /></span>)
                  : <span className="muted">{l.status === 'completed' ? 'Проведён' : 'Отменён'}</span>}
              </div>

              {l.status === 'scheduled' && (
                <form action={saveLessonLink}>
                  <input type="hidden" name="id" value={l.id} />
                  <label>Ссылка на урок<input name="meeting_url" type="url" maxLength={300} placeholder="https://meet.google.com/..." defaultValue={l.meeting_url} /></label>
                  <button className="ghost">Сохранить ссылку</button>
                </form>
              )}

              <form action={saveLessonPlan}>
                <input type="hidden" name="id" value={l.id} />
                <label>План урока<textarea name="plan" rows={3} maxLength={4000} defaultValue={l.plan} /></label>
                <button className="ghost">Сохранить план</button>
              </form>

              {(hw ?? []).filter((h: any) => h.lesson_id === l.id).map((h: any) => (
                <div className="hw" key={h.id}>
                  <p><strong>{h.title}</strong> · {HW[h.status]}</p>
                  {h.body && <p className="muted">{h.body}</p>}
                  {h.status !== 'assigned' && <p>Ответ: {h.answer}</p>}
                  {h.status === 'submitted' && (
                    <form action={checkHomework}>
                      <input type="hidden" name="id" value={h.id} />
                      <label>Комментарий<textarea name="feedback" rows={2} maxLength={4000} /></label>
                      <button className="ghost">Проверено</button>
                    </form>
                  )}
                  {h.status === 'checked' && h.feedback && <p>Комментарий: {h.feedback}</p>}
                  <form action={deleteHomework}><input type="hidden" name="id" value={h.id} /><button className="ghost">Удалить</button></form>
                </div>
              ))}

              <form action={addHomework}>
                <input type="hidden" name="lesson_id" value={l.id} />
                <label>Домашка<input name="title" required maxLength={120} placeholder="Например: задачи 1–5" /></label>
                <label>Подробности<textarea name="body" rows={2} maxLength={4000} /></label>
                <button className="ghost">Задать</button>
              </form>
            </li>
          ))}
        </ul>
      </section>
    </main>
  );
}
```

## `5school/app/tutors/[id]/page.tsx`

```tsx
import { notFound } from 'next/navigation';
import { supabaseServer } from '@/lib/supabase/server';
import { bookLesson } from '@/app/actions';
import { fmt } from '@/lib/time';

type Review = { rating: number; body: string; profiles: { full_name: string } | null };
const stars = (v: number) => '★'.repeat(Math.round(v)) + '☆'.repeat(5 - Math.round(v));

export default async function TutorPage({ params, searchParams }: { params: Promise<{ id: string }>; searchParams: Promise<{ error?: string }> }) {
  const { id } = await params;
  const sp = await searchParams;
  const supabase = await supabaseServer();
  const { data: t } = await supabase
    .from('tutor_profiles')
    .select('*, profiles(full_name)')
    .eq('id', id)
    .eq('is_published', true)
    .eq('is_approved', true)
    .maybeSingle();
  if (!t) notFound();

  const { data: reviews } = await supabase
    .from('reviews')
    .select('rating, body, profiles(full_name)')
    .eq('tutor_id', id)
    .order('created_at', { ascending: false })
    .limit(20);
  const { data: slots } = await supabase
    .from('availability')
    .select('id, starts_at')
    .eq('tutor_id', id)
    .gt('starts_at', new Date().toISOString())
    .order('starts_at')
    .limit(30);
  const list = (reviews ?? []) as unknown as Review[];

  return (
    <main className="wrap">
      <section className="card hero">
        <div>
          <h1>{t.profiles?.full_name}</h1>
          <p className="muted">{t.headline}</p>
          <p><span className="stars">{stars(t.rating_avg)}</span> {t.rating_avg} · {t.rating_count} отзывов</p>
          <div className="tags">{t.subjects.map((s: string) => <span key={s}>{s}</span>)}</div>
        </div>
        <div className="buy"><p className="price">{t.price_per_hour} {t.currency} / час</p></div>
      </section>

      {t.photos.length > 0 && (
        <section className="photos">{t.photos.map((p: string) => <img key={p} src={p} alt="" />)}</section>
      )}

      <section className="card">
        <h2>О себе</h2>
        <p style={{ whiteSpace: 'pre-line', margin: 0 }}>{t.bio || 'Описание пока не добавлено.'}</p>
      </section>

      <section className="card">
        <h2>Свободное время</h2>
        {sp.error && <p className="err">{sp.error}</p>}
        {(slots ?? []).length === 0 && <p className="muted">Свободных слотов пока нет.</p>}
        <div className="slots">
          {(slots ?? []).map((s: any) => (
            <form action={bookLesson} key={s.id}>
              <input type="hidden" name="tutor_id" value={id} />
              <input type="hidden" name="starts_at" value={s.starts_at} />
              <button className="ghost">{fmt(s.starts_at)}</button>
            </form>
          ))}
        </div>
        <p className="muted">Записаться могут ученики. Если вы не вошли, сначала войдите в аккаунт. Время указано по Таллину.</p>
      </section>

      <section>
        <h2>Отзывы</h2>
        {list.length === 0 && <p className="muted">Отзывов пока нет.</p>}
        {list.map((r, i) => (
          <article className="card review" key={i}>
            <p><span className="stars">{stars(r.rating)}</span> <strong>{r.profiles?.full_name}</strong></p>
            {r.body && <p>{r.body}</p>}
          </article>
        ))}
      </section>
    </main>
  );
}
```

## `5school/lib/session.ts`

```ts
import { cookies } from 'next/headers';

// Только для ссылок и кнопок интерфейса. Настоящая проверка входа идёт на сервере в каждой странице.
export async function hasSessionCookie() {
  const store = await cookies();
  return store.getAll().some(
    (c) => c.name.startsWith('sb-') && c.name.includes('-auth-token') && !c.name.endsWith('-code-verifier'),
  );
}
```

## `5school/lib/supabase/server.ts`

```ts
import { createServerClient } from '@supabase/ssr';
import { cookies } from 'next/headers';

export async function supabaseServer() {
  const cookieStore = await cookies();
  return createServerClient(process.env.NEXT_PUBLIC_SUPABASE_URL!, process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!, {
    cookies: {
      getAll: () => cookieStore.getAll(),
      setAll: (list) => {
        try {
          list.forEach(({ name, value, options }) => cookieStore.set(name, value, options));
        } catch {
          // Вызов из Server Component: куки обновит middleware
        }
      },
    },
  });
}
```

## `5school/lib/time.ts`

```ts
// Часовой пояс ввода и отображения времени. Для MVP фиксирован; позже сделаем выбор в профиле.
export const TZ = 'Europe/Tallinn';
export const fmt = (iso: string) =>
  new Date(iso).toLocaleString('ru-RU', { dateStyle: 'medium', timeStyle: 'short', timeZone: TZ });

// На сколько "настенные часы" пояса опережают UTC в данный момент (в мс)
function tzOffset(ts: number, tz: string): number {
  const parts = new Intl.DateTimeFormat('en-US', {
    timeZone: tz,
    hourCycle: 'h23',
    year: 'numeric', month: '2-digit', day: '2-digit', hour: '2-digit', minute: '2-digit', second: '2-digit',
  }).formatToParts(new Date(ts));
  const get = (type: string) => Number(parts.find((p) => p.type === type)!.value);
  return Date.UTC(get('year'), get('month') - 1, get('day'), get('hour'), get('minute'), get('second')) - ts;
}

// "2026-10-20T14:00" (время в поясе tz) -> ISO в UTC. Если формат неверный, вернёт null.
export function zonedToIso(local: string, tz: string = TZ): string | null {
  const m = /^(\d{4})-(\d{2})-(\d{2})T(\d{2}):(\d{2})(?::\d{2})?$/.exec(local);
  if (!m) return null;
  const [y, mo, d, h, mi] = m.slice(1, 6).map(Number);
  const wall = Date.UTC(y, mo - 1, d, h, mi);
  let ts = wall - tzOffset(wall, tz);
  ts = wall - tzOffset(ts, tz); // второй проход учитывает переход на летнее/зимнее время
  const date = new Date(ts);
  return Number.isNaN(date.getTime()) ? null : date.toISOString();
}
```

## `5school/middleware.ts`

```ts
import { createServerClient } from '@supabase/ssr';
import { NextResponse, type NextRequest } from 'next/server';

const PROTECTED = ['/dashboard', '/tutor/profile', '/tutor/schedule'];

export async function middleware(req: NextRequest) {
  let res = NextResponse.next({ request: req });
  const supabase = createServerClient(process.env.NEXT_PUBLIC_SUPABASE_URL!, process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!, {
    cookies: {
      getAll: () => req.cookies.getAll(),
      setAll: (list) => {
        list.forEach(({ name, value }) => req.cookies.set(name, value));
        res = NextResponse.next({ request: req });
        list.forEach(({ name, value, options }) => res.cookies.set(name, value, options));
      },
    },
  });
  const { data: { user } } = await supabase.auth.getUser();
  const needsAuth = PROTECTED.some((p) => req.nextUrl.pathname.startsWith(p));
  if (!user && needsAuth) return NextResponse.redirect(new URL('/login', req.url));
  return res;
}

export const config = { matcher: ['/((?!_next|favicon.ico|.*\\.(?:png|jpg|jpeg|svg|webp)$).*)'] };
```

## `5school/next.config.ts`

```ts
import type { NextConfig } from 'next';

// По умолчанию Server Actions принимают тело до 1 МБ, а фото могут быть до 4 МБ
const config: NextConfig = { experimental: { serverActions: { bodySizeLimit: '5mb' } } };
export default config;
```

## `5school/package.json`

```json
{
  "name": "5school",
  "private": true,
  "scripts": { "dev": "next dev", "build": "next build", "start": "next start" },
  "dependencies": {
    "next": "^15.1.0",
    "react": "^19.0.0",
    "react-dom": "^19.0.0",
    "@supabase/supabase-js": "^2.45.0",
    "@supabase/ssr": "^0.5.0"
  },
  "devDependencies": {
    "typescript": "^5.6.0",
    "@types/node": "^22.0.0",
    "@types/react": "^19.0.0",
    "@types/react-dom": "^19.0.0"
  }
}
```

## `5school/supabase/schema.sql`

```sql
-- 5school: схема базы. Вставить в Supabase → SQL Editor и выполнить целиком.
create type user_role as enum ('student', 'tutor');

create table profiles (
  id uuid primary key references auth.users on delete cascade,
  role user_role not null,
  full_name text not null,
  created_at timestamptz not null default now()
);

create table tutor_profiles (
  id uuid primary key references profiles on delete cascade,
  headline text not null default '',
  bio text not null default '',
  subjects text[] not null default '{}',
  price_per_hour integer not null default 0 check (price_per_hour >= 0),
  currency text not null default 'EUR',
  photos text[] not null default '{}',
  rating_avg numeric(2,1) not null default 0,
  rating_count integer not null default 0,
  is_published boolean not null default false,
  updated_at timestamptz not null default now()
);

create table lessons (
  id uuid primary key default gen_random_uuid(),
  tutor_id uuid not null references tutor_profiles on delete cascade,
  student_id uuid not null references profiles on delete cascade,
  starts_at timestamptz not null,
  status text not null default 'scheduled' check (status in ('scheduled', 'completed', 'cancelled')),
  created_at timestamptz not null default now()
);

create table reviews (
  id uuid primary key default gen_random_uuid(),
  tutor_id uuid not null references tutor_profiles on delete cascade,
  student_id uuid not null references profiles on delete cascade,
  rating smallint not null check (rating between 1 and 5),
  body text not null default '' check (char_length(body) <= 1000),
  created_at timestamptz not null default now(),
  unique (tutor_id, student_id)
);

-- Общая оценка пересчитывается при каждом изменении отзывов
create or replace function refresh_rating() returns trigger language plpgsql security definer set search_path = public as $$
begin
  update tutor_profiles t set
    rating_count = (select count(*) from reviews r where r.tutor_id = t.id),
    rating_avg = coalesce((select round(avg(r.rating), 1) from reviews r where r.tutor_id = t.id), 0)
  where t.id = coalesce(new.tutor_id, old.tutor_id);
  return null;
end $$;
create trigger reviews_rating after insert or update or delete on reviews
  for each row execute function refresh_rating();

-- При регистрации создаём профиль; роль и имя берём из metadata
create or replace function handle_new_user() returns trigger language plpgsql security definer set search_path = public as $$
begin
  insert into profiles (id, role, full_name)
  values (
    new.id,
    coalesce((new.raw_user_meta_data->>'role')::user_role, 'student'),
    coalesce(new.raw_user_meta_data->>'full_name', 'Пользователь')
  );
  if new.raw_user_meta_data->>'role' = 'tutor' then
    insert into tutor_profiles (id) values (new.id);
  end if;
  return new;
end $$;
create trigger on_auth_user_created after insert on auth.users
  for each row execute function handle_new_user();

-- Безопасность (RLS)
alter table profiles enable row level security;
alter table tutor_profiles enable row level security;
alter table lessons enable row level security;
alter table reviews enable row level security;

create policy "profiles: читать" on profiles for select using (true);

create policy "tutors: видны опубликованные" on tutor_profiles for select
  using (is_published or auth.uid() = id);
create policy "tutors: правит только себя" on tutor_profiles for update
  using (auth.uid() = id) with check (auth.uid() = id);
-- Рейтинг нельзя подменить вручную: обновлять можно только поля анкеты
revoke update on tutor_profiles from authenticated;
grant update (headline, bio, subjects, price_per_hour, photos, is_published, updated_at) on tutor_profiles to authenticated;

create policy "lessons: видят участники" on lessons for select
  using (auth.uid() = student_id or auth.uid() = tutor_id);
create policy "lessons: ученик записывается" on lessons for insert
  with check (auth.uid() = student_id and status = 'scheduled');
create policy "lessons: репетитор отмечает итог" on lessons for update
  using (auth.uid() = tutor_id) with check (auth.uid() = tutor_id);

create policy "reviews: читают все" on reviews for select using (true);
-- Оставить отзыв можно только после завершённого урока с этим репетитором
create policy "reviews: после урока" on reviews for insert
  with check (
    auth.uid() = student_id and exists (
      select 1 from lessons l
      where l.tutor_id = reviews.tutor_id and l.student_id = auth.uid() and l.status = 'completed'
    )
  );
create policy "reviews: правит автор" on reviews for update
  using (auth.uid() = student_id) with check (auth.uid() = student_id);
```

## `5school/supabase/schema_stage3.sql`

```sql
-- 5school, этап 3: расписание, запись и фото. Выполнить ПОСЛЕ schema.sql.
create table availability (
  id uuid primary key default gen_random_uuid(),
  tutor_id uuid not null references tutor_profiles on delete cascade,
  starts_at timestamptz not null,
  created_at timestamptz not null default now(),
  unique (tutor_id, starts_at)
);
alter table availability enable row level security;
create policy "slots: читают все" on availability for select using (true);
create policy "slots: репетитор добавляет" on availability for insert
  with check (auth.uid() = tutor_id and starts_at > now());
create policy "slots: репетитор удаляет свои" on availability for delete
  using (auth.uid() = tutor_id);

-- Запись одним действием: слот удаляется и создаётся урок
create or replace function book_lesson(p_tutor uuid, p_starts timestamptz)
returns uuid language plpgsql security definer set search_path = public as $$
declare v_lesson uuid;
begin
  if auth.uid() is null then raise exception 'not_authenticated'; end if;
  if not exists (select 1 from profiles where id = auth.uid() and role = 'student') then
    raise exception 'only_students_can_book';
  end if;
  delete from availability where tutor_id = p_tutor and starts_at = p_starts;
  if not found then raise exception 'slot_taken'; end if;
  insert into lessons (tutor_id, student_id, starts_at) values (p_tutor, auth.uid(), p_starts)
  returning id into v_lesson;
  return v_lesson;
end $$;
revoke execute on function book_lesson(uuid, timestamptz) from public;
grant execute on function book_lesson(uuid, timestamptz) to authenticated;

-- Фото репетиторов: публичное чтение, загрузка только в свою папку
insert into storage.buckets (id, name, public) values ('tutor-photos', 'tutor-photos', true)
  on conflict (id) do nothing;
create policy "photos: читают все" on storage.objects for select
  using (bucket_id = 'tutor-photos');
create policy "photos: загружает владелец" on storage.objects for insert to authenticated
  with check (bucket_id = 'tutor-photos' and (storage.foldername(name))[1] = auth.uid()::text);
create policy "photos: удаляет владелец" on storage.objects for delete to authenticated
  using (bucket_id = 'tutor-photos' and (storage.foldername(name))[1] = auth.uid()::text);
```

## `5school/supabase/schema_stage4.sql`

```sql
-- 5school, этап 4: планы уроков и домашка. Выполнить ПОСЛЕ schema_stage3.sql.
alter table lessons add column plan text not null default '' check (char_length(plan) <= 4000);

create table homework (
  id uuid primary key default gen_random_uuid(),
  lesson_id uuid not null references lessons on delete cascade,
  title text not null check (char_length(title) between 1 and 120),
  body text not null default '' check (char_length(body) <= 4000),
  status text not null default 'assigned' check (status in ('assigned', 'submitted', 'checked')),
  answer text not null default '' check (char_length(answer) <= 4000),
  feedback text not null default '' check (char_length(feedback) <= 4000),
  created_at timestamptz not null default now()
);
alter table homework enable row level security;

-- Видят домашку только участники урока
create policy "homework: видят участники урока" on homework for select using (
  exists (select 1 from lessons l where l.id = homework.lesson_id and (l.student_id = auth.uid() or l.tutor_id = auth.uid()))
);
-- Задаёт и удаляет только репетитор этого урока
create policy "homework: репетитор задаёт" on homework for insert with check (
  exists (select 1 from lessons l where l.id = homework.lesson_id and l.tutor_id = auth.uid())
);
create policy "homework: репетитор удаляет" on homework for delete using (
  exists (select 1 from lessons l where l.id = homework.lesson_id and l.tutor_id = auth.uid())
);

-- Ответ и проверку меняем только через функции, чтобы ученик не мог поставить себе "проверено"
revoke update on homework from authenticated;

create or replace function submit_homework(p_id uuid, p_answer text) returns void
language plpgsql security definer set search_path = public as $$
begin
  update homework h set answer = left(p_answer, 4000), status = 'submitted'
  from lessons l
  where h.id = p_id and l.id = h.lesson_id and l.student_id = auth.uid() and h.status = 'assigned';
  if not found then raise exception 'not_allowed'; end if;
end $$;

create or replace function check_homework(p_id uuid, p_feedback text) returns void
language plpgsql security definer set search_path = public as $$
begin
  update homework h set feedback = left(p_feedback, 4000), status = 'checked'
  from lessons l
  where h.id = p_id and l.id = h.lesson_id and l.tutor_id = auth.uid() and h.status = 'submitted';
  if not found then raise exception 'not_allowed'; end if;
end $$;

grant execute on function submit_homework(uuid, text) to authenticated;
grant execute on function check_homework(uuid, text) to authenticated;
```

## `5school/supabase/schema_stage5.sql`

```sql
-- 5school, этап 5: исправления безопасности. Выполнить ПОСЛЕ schema_stage4.sql.
-- Файл можно запускать повторно.

-- 1. Записаться на урок можно только через book_lesson (проверяет слот, двойной записи нет)
drop policy if exists "lessons: ученик записывается" on lessons;
revoke insert on lessons from authenticated, anon;

-- 2. Репетитор меняет в уроке только план. Итог урока ставит функция complete_lesson.
revoke update on lessons from authenticated, anon;
grant update (plan) on lessons to authenticated;

create or replace function complete_lesson(p_id uuid) returns void
language plpgsql security definer set search_path = public as $$
begin
  update lessons set status = 'completed'
  where id = p_id and tutor_id = auth.uid() and status = 'scheduled' and starts_at <= now();
  if not found then raise exception 'not_allowed'; end if;
end $$;
revoke execute on function complete_lesson(uuid) from public, anon;
grant execute on function complete_lesson(uuid) to authenticated;

-- 3. Отзыв нельзя переселить к другому репетитору или от другого ученика
create or replace function reviews_lock_keys() returns trigger language plpgsql as $$
begin
  if new.tutor_id <> old.tutor_id or new.student_id <> old.student_id then
    raise exception 'review_keys_locked';
  end if;
  return new;
end $$;
drop trigger if exists reviews_keys_locked on reviews;
create trigger reviews_keys_locked before update on reviews
  for each row execute function reviews_lock_keys();

-- 4. Ученик видит анкеты репетиторов, у которых записан, даже если анкета снята с публикации
drop policy if exists "tutors: видны опубликованные" on tutor_profiles;
drop policy if exists "tutors: видны опубликованные и по урокам" on tutor_profiles;
create policy "tutors: видны опубликованные и по урокам" on tutor_profiles for select
  using (
    is_published or auth.uid() = id
    or exists (select 1 from lessons l where l.tutor_id = tutor_profiles.id and l.student_id = auth.uid())
  );

-- 5. Фото: колонку photos напрямую менять нельзя, только через функцию.
--    Она пропускает лишь ссылки на Storage в папке самого репетитора (не больше 10).
revoke update on tutor_profiles from authenticated;
grant update (headline, bio, subjects, price_per_hour, is_published, updated_at) on tutor_profiles to authenticated;

create or replace function set_my_photos(p_photos text[]) returns void
language plpgsql security definer set search_path = public as $$
begin
  if auth.uid() is null then raise exception 'not_authenticated'; end if;
  if coalesce(array_length(p_photos, 1), 0) > 10 then raise exception 'too_many'; end if;
  if exists (
    select 1 from unnest(p_photos) as u(url)
    where u.url !~ ('^https?://[^/]+/storage/v1/object/public/tutor-photos/' || auth.uid()::text || '/[0-9a-f-]{36}$')
  ) then raise exception 'bad_photo'; end if;
  update tutor_profiles set photos = p_photos, updated_at = now() where id = auth.uid();
  if not found then raise exception 'not_a_tutor'; end if;
end $$;
revoke execute on function set_my_photos(text[]) from public, anon;
grant execute on function set_my_photos(text[]) to authenticated;

-- 6. Бакет с фото: только jpeg/png/webp и не больше 4 МБ
update storage.buckets
  set file_size_limit = 4194304, allowed_mime_types = array['image/jpeg', 'image/png', 'image/webp']
  where id = 'tutor-photos';

-- 7. Запись: слот должен быть в будущем, а анкета опубликована
create or replace function book_lesson(p_tutor uuid, p_starts timestamptz)
returns uuid language plpgsql security definer set search_path = public as $$
declare v_lesson uuid;
begin
  if auth.uid() is null then raise exception 'not_authenticated'; end if;
  if not exists (select 1 from profiles where id = auth.uid() and role = 'student') then
    raise exception 'only_students_can_book';
  end if;
  if p_starts <= now() then raise exception 'slot_in_past'; end if;
  if not exists (select 1 from tutor_profiles where id = p_tutor and is_published) then
    raise exception 'tutor_not_available';
  end if;
  delete from availability where tutor_id = p_tutor and starts_at = p_starts;
  if not found then raise exception 'slot_taken'; end if;
  insert into lessons (tutor_id, student_id, starts_at) values (p_tutor, auth.uid(), p_starts)
  returning id into v_lesson;
  return v_lesson;
end $$;
revoke execute on function book_lesson(uuid, timestamptz) from public, anon;
grant execute on function book_lesson(uuid, timestamptz) to authenticated;
```

## `5school/supabase/schema_stage6.sql`

```sql
-- 5school, этап 6: ссылка на урок, отмена урока, одобрение анкет. Выполнить ПОСЛЕ schema_stage5.sql.
-- Файл можно запускать повторно.

-- 1. Ссылка на урок хранится в уроке, а не в анкете: её видят только ученик и репетитор этого урока
alter table lessons add column if not exists meeting_url text not null default ''
  check (meeting_url = '' or (meeting_url ~ '^https://[^[:space:]]+$' and char_length(meeting_url) <= 300));
revoke update on lessons from authenticated, anon;
grant update (plan, meeting_url) on lessons to authenticated;

-- 2. Отмена урока до его начала (ученик или репетитор).
--    Слот возвращается в расписание, только если отменил ученик.
create or replace function cancel_lesson(p_id uuid) returns void
language plpgsql security definer set search_path = public as $$
declare v lessons%rowtype;
begin
  select * into v from lessons
  where id = p_id and status = 'scheduled' and starts_at > now()
    and (student_id = auth.uid() or tutor_id = auth.uid())
  for update;
  if not found then raise exception 'not_allowed'; end if;
  update lessons set status = 'cancelled' where id = p_id;
  if v.student_id = auth.uid() then
    insert into availability (tutor_id, starts_at) values (v.tutor_id, v.starts_at)
    on conflict (tutor_id, starts_at) do nothing;
  end if;
end $$;
revoke execute on function cancel_lesson(uuid) from public, anon;
grant execute on function cancel_lesson(uuid) to authenticated;

-- 3. Одобрение анкет вручную: в каталоге видны только опубликованные И одобренные.
--    Репетитор не может одобрить себя: колонки is_approved нет в списке grant update.
--    Уже существующие анкеты одобряются один раз, при добавлении колонки.
do $$
begin
  if not exists (
    select 1 from information_schema.columns
    where table_schema = 'public' and table_name = 'tutor_profiles' and column_name = 'is_approved'
  ) then
    alter table tutor_profiles add column is_approved boolean not null default false;
    update tutor_profiles set is_approved = true;
  end if;
end $$;

drop policy if exists "tutors: видны опубликованные и по урокам" on tutor_profiles;
drop policy if exists "tutors: видны одобренные и по урокам" on tutor_profiles;
create policy "tutors: видны одобренные и по урокам" on tutor_profiles for select
  using (
    (is_published and is_approved) or auth.uid() = id
    or exists (select 1 from lessons l where l.tutor_id = tutor_profiles.id and l.student_id = auth.uid())
  );

-- Запись возможна только к одобренному и опубликованному репетитору
create or replace function book_lesson(p_tutor uuid, p_starts timestamptz)
returns uuid language plpgsql security definer set search_path = public as $$
declare v_lesson uuid;
begin
  if auth.uid() is null then raise exception 'not_authenticated'; end if;
  if not exists (select 1 from profiles where id = auth.uid() and role = 'student') then
    raise exception 'only_students_can_book';
  end if;
  if p_starts <= now() then raise exception 'slot_in_past'; end if;
  if not exists (select 1 from tutor_profiles where id = p_tutor and is_published and is_approved) then
    raise exception 'tutor_not_available';
  end if;
  delete from availability where tutor_id = p_tutor and starts_at = p_starts;
  if not found then raise exception 'slot_taken'; end if;
  insert into lessons (tutor_id, student_id, starts_at) values (p_tutor, auth.uid(), p_starts)
  returning id into v_lesson;
  return v_lesson;
end $$;
revoke execute on function book_lesson(uuid, timestamptz) from public, anon;
grant execute on function book_lesson(uuid, timestamptz) to authenticated;
```

## `5school/tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "ES2022", "lib": ["dom", "dom.iterable", "esnext"], "allowJs": true,
    "skipLibCheck": true, "strict": true, "noEmit": true, "esModuleInterop": true,
    "module": "esnext", "moduleResolution": "bundler", "resolveJsonModule": true,
    "isolatedModules": true, "jsx": "preserve", "incremental": true,
    "plugins": [{ "name": "next" }], "paths": { "@/*": ["./*"] }
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts"],
  "exclude": ["node_modules"]
}
```

