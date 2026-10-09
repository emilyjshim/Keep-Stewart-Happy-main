# Keep Stewart Happy

The game uses Supabase so everyone shares Stewart's needs, mood, points, turns, and level.
Needs gradually decrease while nobody is caring for Stewart: fullness by 12 points per
hour, energy by 10 points per hour, and fun by 14 points per hour.

## Connect the shared game

1. Create a Supabase project at [supabase.com](https://supabase.com/).
2. In the Supabase dashboard, open **SQL Editor**, create a query, paste the SQL below,
   and run it.
3. In **Project Settings → API**, copy the **Project URL** and the **anon/public key**.
   Never use or publish the `service_role` key in this website.
4. In `index.html`, replace `YOUR_SUPABASE_URL` and `YOUR_SUPABASE_ANON_KEY` with those
   two values.
5. In **Database → Replication**, enable Realtime for the `stewart_game` table. The page
   also checks for updates every 30 seconds if Realtime is temporarily unavailable.
6. Host the project on a static web host and open the hosted page. The Supabase JavaScript
   library is loaded from jsDelivr, so players need an internet connection.

The anon key is intended to be used in public websites. Row-level security below allows
anyone to read the shared game, but only the validated database functions can change it.
Because this is a shared public game, anyone with the page can care for Stewart.

```sql
create table if not exists public.stewart_game (
  id smallint primary key check (id = 1),
  fullness numeric not null check (fullness between 0 and 100),
  energy numeric not null check (energy between 0 and 100),
  fun numeric not null check (fun between 0 and 100),
  level integer not null check (level >= 1),
  points integer not null check (points >= 0),
  turns integer not null check (turns >= 0),
  goal_target integer not null check (goal_target between 60 and 90),
  is_complete boolean not null default false,
  updated_at timestamptz not null default now()
);

alter table public.stewart_game enable row level security;

drop policy if exists "Anyone can read the shared Stewart game" on public.stewart_game;
create policy "Anyone can read the shared Stewart game"
  on public.stewart_game for select to anon, authenticated
  using (true);

revoke insert, update, delete on public.stewart_game from anon, authenticated;
grant select on public.stewart_game to anon, authenticated;

create or replace function public.get_stewart_state()
returns jsonb
language plpgsql
security definer
set search_path = public, pg_temp
as $$
declare
  v_game public.stewart_game%rowtype;
  v_elapsed_hours numeric;
begin
  insert into public.stewart_game
    (id, fullness, energy, fun, level, points, turns, goal_target, is_complete, updated_at)
  values (1, 20, 35, 25, 1, 0, 0, 60, false, clock_timestamp())
  on conflict (id) do nothing;

  select * into v_game from public.stewart_game where id = 1;
  v_elapsed_hours := greatest(
    extract(epoch from (clock_timestamp() - v_game.updated_at)),
    0
  ) / 3600.0;

  return jsonb_build_object(
    'needs', jsonb_build_object(
      'fullness', round(greatest(0::numeric, v_game.fullness - 12 * v_elapsed_hours), 1),
      'energy', round(greatest(0::numeric, v_game.energy - 10 * v_elapsed_hours), 1),
      'fun', round(greatest(0::numeric, v_game.fun - 14 * v_elapsed_hours), 1)
    ),
    'level', v_game.level,
    'points', v_game.points,
    'turns', v_game.turns,
    'goal_target', v_game.goal_target,
    'complete', v_game.is_complete
  );
end;
$$;

create or replace function public.care_for_stewart(p_action text)
returns jsonb
language plpgsql
security definer
set search_path = public, pg_temp
as $$
declare
  v_game public.stewart_game%rowtype;
  v_elapsed_hours numeric;
  v_fullness_change numeric;
  v_energy_change numeric;
  v_fun_change numeric;
  v_message text;
  v_ready_count integer;
begin
  case p_action
    when 'food' then
      v_fullness_change := 14;
      v_energy_change := 2;
      v_fun_change := 2;
      v_message := 'Yum! Stewart enjoyed his food. +5 happy points.';
    when 'play' then
      v_fullness_change := 0;
      v_energy_change := -4;
      v_fun_change := 12;
      v_message := 'Stewart had fun playing with you! +5 happy points.';
    when 'poppo' then
      v_fullness_change := 3;
      v_energy_change := 8;
      v_fun_change := 10;
      v_message := 'Stewart loved spending time with Poppo! +5 happy points.';
    else
      raise exception 'Unknown care action';
  end case;

  insert into public.stewart_game
    (id, fullness, energy, fun, level, points, turns, goal_target, is_complete, updated_at)
  values (1, 20, 35, 25, 1, 0, 0, 60, false, clock_timestamp())
  on conflict (id) do nothing;

  select * into v_game from public.stewart_game where id = 1 for update;
  v_elapsed_hours := greatest(
    extract(epoch from (clock_timestamp() - v_game.updated_at)),
    0
  ) / 3600.0;

  v_game.fullness := greatest(0::numeric, least(100::numeric,
    v_game.fullness - 12 * v_elapsed_hours + v_fullness_change));
  v_game.energy := greatest(0::numeric, least(100::numeric,
    v_game.energy - 10 * v_elapsed_hours + v_energy_change));
  v_game.fun := greatest(0::numeric, least(100::numeric,
    v_game.fun - 14 * v_elapsed_hours + v_fun_change));
  v_game.points := v_game.points + 5;
  v_game.turns := v_game.turns + 1;

  v_ready_count :=
    (case when v_game.fullness >= v_game.goal_target then 1 else 0 end) +
    (case when v_game.energy >= v_game.goal_target then 1 else 0 end) +
    (case when v_game.fun >= v_game.goal_target then 1 else 0 end);

  if v_ready_count = 3 and not v_game.is_complete then
    v_game.points := v_game.points + 25;
    if v_game.goal_target >= 90 then
      v_game.is_complete := true;
      v_message := 'You did it! Stewart is feeling his very best. You earned 25 bonus points!';
    else
      v_game.level := v_game.level + 1;
      v_game.goal_target := least(v_game.goal_target + 10, 90);
      v_message := 'Care goal complete! You earned 25 bonus points. The next level is ready!';
    end if;
  end if;

  update public.stewart_game
  set fullness = v_game.fullness,
      energy = v_game.energy,
      fun = v_game.fun,
      level = v_game.level,
      points = v_game.points,
      turns = v_game.turns,
      goal_target = v_game.goal_target,
      is_complete = v_game.is_complete,
      updated_at = clock_timestamp()
  where id = 1;

  return jsonb_build_object(
    'needs', jsonb_build_object(
      'fullness', round(v_game.fullness, 1),
      'energy', round(v_game.energy, 1),
      'fun', round(v_game.fun, 1)
    ),
    'level', v_game.level,
    'points', v_game.points,
    'turns', v_game.turns,
    'goal_target', v_game.goal_target,
    'complete', v_game.is_complete,
    'message', v_message
  );
end;
$$;

revoke all on function public.get_stewart_state() from public, anon, authenticated;
revoke all on function public.care_for_stewart(text) from public, anon, authenticated;
grant execute on function public.get_stewart_state() to anon, authenticated;
grant execute on function public.care_for_stewart(text) to anon, authenticated;
```
