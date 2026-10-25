# class-anthem-vote-
-- 班级合唱投票数据库初始化
create table if not exists public.anthem_votes (
  voter_id uuid primary key,
  choice text not null check (choice in ('A','B','C')),
  suggestion_name text,
  suggestion_url text,
  suggestion_reason text,
  created_at timestamptz not null default now(),
  constraint c_suggestion_name_required
    check (
      choice <> 'C'
      or (
        suggestion_name is not null
        and length(trim(suggestion_name)) > 0
      )
    )
);

alter table public.anthem_votes enable row level security;

revoke all on table public.anthem_votes from anon, authenticated;
grant usage on schema public to anon, authenticated;

create or replace function public.cast_vote(
  p_choice text,
  p_suggestion jsonb default null
)
returns jsonb
language plpgsql
security definer
set search_path = ''
as $$
declare
  v_uid uuid := auth.uid();
  v_name text;
  v_url text;
  v_reason text;
begin
  if v_uid is null then
    raise exception '请先建立匿名会话后再投票';
  end if;

  if p_choice not in ('A','B','C') then
    raise exception '无效的投票选项';
  end if;

  if p_choice = 'C' then
    v_name := left(trim(coalesce(p_suggestion->>'name','')),100);
    v_url := left(trim(coalesce(p_suggestion->>'url','')),500);
    v_reason := left(trim(coalesce(p_suggestion->>'reason','')),240);

    if v_name = '' then
      raise exception '请填写推荐版本名称';
    end if;

    if v_url <> '' and v_url !~* '^https?://' then
      raise exception '试听链接格式无效';
    end if;
  end if;

  insert into public.anthem_votes(
    voter_id, choice, suggestion_name,
    suggestion_url, suggestion_reason
  )
  values (
    v_uid, p_choice,
    case when p_choice='C' then v_name else null end,
    case when p_choice='C' and v_url<>'' then v_url else null end,
    case when p_choice='C' and v_reason<>'' then v_reason else null end
  );

  return jsonb_build_object('ok',true);

exception
  when unique_violation then
    return jsonb_build_object(
      'ok',false,
      'message','这个匿名会话已经投过票了'
    );
end;
$$;

create or replace function public.has_my_vote()
returns jsonb
language plpgsql
security definer
set search_path = ''
as $$
declare
  v_uid uuid := auth.uid();
  v_found boolean := false;
begin
  if v_uid is null then
    return jsonb_build_object('has_voted',false);
  end if;

  select exists(
    select 1
    from public.anthem_votes
    where voter_id = v_uid
  ) into v_found;

  return jsonb_build_object('has_voted',v_found);
end;
$$;

create or replace function public.get_vote_results()
returns jsonb
language plpgsql
security definer
set search_path = ''
as $$
declare
  v_a integer;
  v_b integer;
  v_c integer;
  v_suggestions jsonb;
begin
  select
    count(*) filter (where choice='A'),
    count(*) filter (where choice='B'),
    count(*) filter (where choice='C')
  into v_a,v_b,v_c
  from public.anthem_votes;

  select coalesce(
    jsonb_agg(
      jsonb_build_object(
        'name',suggestion_name,
        'url',suggestion_url,
        'reason',suggestion_reason,
        'created_at',created_at
      ) order by created_at asc
    ),
    '[]'::jsonb
  )
  into v_suggestions
  from public.anthem_votes
  where choice='C' and suggestion_name is not null;

  return jsonb_build_object(
    'a_count',v_a,
    'b_count',v_b,
    'c_count',v_c,
    'suggestions',v_suggestions
  );
end;
$$;

revoke all on function public.cast_vote(text,jsonb) from public;
revoke all on function public.has_my_vote() from public;
revoke all on function public.get_vote_results() from public;

grant execute on function public.cast_vote(text,jsonb)
  to anon, authenticated;
grant execute on function public.has_my_vote()
  to anon, authenticated;
grant execute on function public.get_vote_results()
  to anon, authenticated;

-- 不向普通访客开放投票表的直接读取权限