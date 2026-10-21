# Keep Stewart Happy

The game uses Supabase so everyone shares Stewart's needs, mood, points, turns, and level.
Needs gradually decrease while nobody is caring for Stewart: fullness by 12 points per
hour, energy by 10 points per hour, and fun by 14 points per hour.

The care screen shows each need as a percentage and level bar. **Food** opens the kitchen,
where players can tap a food or drag it to Stewart. His shared craving determines which foods he'll
accept; the craving cycles through sweet, thirsty, and hungry after a successful serving.
Beet juice and ice cream are included. When Stewart is very hungry and has 5% fullness
or less, a
flashing **FEED ME** warning appears; at 100%, Stewart switches to his full artwork and
refuses more food.

If all available foods for his current craving have been served during a kitchen visit,
you can buy and serve an extra matching portion for 10 happy points. The purchase and
serving happen together in the shared save; you cannot buy food when you have fewer than
10 points.

Soccer lasts up to 30 seconds. Stewart gets faster at 10 seconds ("Stewart is now
warmed up!") and again at 20 seconds ("Stewart really wants to win!"). The faster ball
changes direction more sharply, so time your moves to return it. Finishing soccer
increases Stewart's fun by 12 points; the care screen confirms the increase.

**School Run** opens a driving mini-game: steer between three lanes with the left and
right arrow keys (or the on-screen buttons), and hold the up arrow to accelerate. Avoid
road barriers and collect stars on the way to school. A collision stops the drive and
offers a retry. Completing the drive counts as a Play care action in the shared game;
leaving early does not count.

Each accepted serving lowers cleanliness by 55 points, so Stewart's teeth look dirty
after he eats. Open the bathroom, drag toothpaste onto the toothbrush, then scrub back
and forth over his teeth until they're clean. Brushing restores cleanliness to 100%.

## Preview low moods without changing the shared save

To preview Stewart with critical needs and the **FEED ME** warning, open the page with
`?testMood=low` appended to its URL (for example, `index.html?testMood=low`). This
browser-only preview displays fullness at 4%, energy at 8%, fun at 6%, cleanliness at
35%, and a hungry craving. Brushing and soccer fun increases work locally in the
preview, while other care actions and feeding are prevented from updating Supabase.
Remove the query parameter to return to the actual shared game state.

## Connect the shared game

1. Use your existing Supabase project at [supabase.com](https://supabase.com/); no second project is needed.
2. In the Supabase dashboard, open **SQL Editor**, create a query, paste the SQL below,
   and run it. Then run the **Add shared cleanliness and dental care** migration at the
   end of this document in a separate query.
3. In **Project Settings → API**, copy the **Project URL** and the **anon/public key**.
   Never use or publish the `service_role` key in this website.
4. Ensure `SUPABASE_URL` and `SUPABASE_ANON_KEY` in `index.html` contain that project's
   Project URL and anon/publishable key. They are already filled in for the current setup.
5. In **Database → Replication**, enable Realtime for the `stewart_game` table. The page
   also checks for updates every 30 seconds if Realtime is temporarily unavailable.
6. Host the project on a static web host and open the hosted page. The Supabase JavaScript
   library is loaded from jsDelivr, so players need an internet connection.

The anon key is intended to be used in public websites. Row-level security below allows
anyone to read the shared game, but only the validated database functions can change it.
Because this is a shared public game, anyone with the page can care for Stewart.
The Emily/Evan chooser is a name selector, not password-protected authentication.

## Fix an existing table missing game columns

If you already ran the setup and the game reports an error such as
`column "fullness" of relation "stewart_game" does not exist`, the table existed before
the setup script. `CREATE TABLE IF NOT EXISTS` leaves an existing table unchanged. Run
this additive migration in a **new Supabase SQL Editor query**. It keeps existing rows
and adds any missing game columns with starting values:

```sql
alter table public.stewart_game
  add column if not exists fullness numeric not null default 20 check (fullness between 0 and 100),
  add column if not exists energy numeric not null default 35 check (energy between 0 and 100),
  add column if not exists fun numeric not null default 25 check (fun between 0 and 100),
  add column if not exists level integer not null default 1 check (level >= 1),
  add column if not exists points integer not null default 0 check (points >= 0),
  add column if not exists turns integer not null default 0 check (turns >= 0),
  add column if not exists goal_target integer not null default 60 check (goal_target between 60 and 90),
  add column if not exists is_complete boolean not null default false,
  add column if not exists updated_at timestamptz not null default now();
```

After it completes successfully, reload the game and choose **Let's go!** again. If
Supabase reports a different missing column, the existing table may have a different
layout; share the exact error rather than deleting or resetting the table.

```sql
create table if not exists public.stewart_game (
  id smallint primary key check (id = 1),
  fullness numeric not null check (fullness between 0 and 100),
  energy numeric not null check (energy between 0 and 100),
  fun numeric not null check (fun between 0 and 100),
  level integer not null check (level >= 1),
  points integer not null check (points >= 0),
  turns integer not null check (turns >= 0),
  emily_care_count integer not null default 0 check (emily_care_count >= 0),
  evan_care_count integer not null default 0 check (evan_care_count >= 0),
  food_craving text not null default 'sweet' check (food_craving in ('sweet', 'thirsty', 'hungry')),
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
    'food_craving', v_game.food_craving,
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

drop function if exists public.care_for_stewart_by_player(text, text);
create or replace function public.care_for_stewart_by_player(
  p_action text,
  p_player text,
  p_food text default null
)
returns jsonb
language plpgsql
security definer
set search_path = public, pg_temp
as $$
declare
  v_result jsonb;
  v_game public.stewart_game%rowtype;
  v_elapsed_hours numeric;
  v_fullness numeric;
  v_next_craving text;
  v_emily_count integer;
  v_evan_count integer;
begin
  if p_player is null or p_player not in ('Emily', 'Evan') then
    raise exception 'Unknown player';
  end if;

  if p_action = 'food' then
    select * into v_game from public.stewart_game where id = 1 for update;
    v_elapsed_hours := greatest(
      extract(epoch from (clock_timestamp() - v_game.updated_at)),
      0
    ) / 3600.0;
    v_fullness := round(greatest(0::numeric, v_game.fullness - 12 * v_elapsed_hours), 1);

    if round(v_fullness) >= 100 then
      return jsonb_build_object(
        'accepted', false,
        'food_craving', v_game.food_craving,
        'message', 'Stewart is too full to eat right now.'
      );
    end if;

    if p_food is null or p_food not in ('galbi', 'fish', 'steak', 'msr', 'thai_tea', 'roll', 'beet_juice', 'ice_cream') then
      raise exception 'Unknown food item';
    end if;

    if (v_game.food_craving = 'sweet' and p_food not in ('ice_cream', 'msr', 'roll', 'thai_tea'))
      or (v_game.food_craving = 'thirsty' and p_food not in ('thai_tea', 'beet_juice'))
      or (v_game.food_craving = 'hungry' and p_food not in ('galbi', 'fish', 'steak')) then
      return jsonb_build_object(
        'accepted', false,
        'food_craving', v_game.food_craving,
        'message', case v_game.food_craving
          when 'sweet' then 'Stewart wants something sweet. Try ice cream, mango sticky rice, a cinnamon roll, or Thai tea.'
          when 'thirsty' then 'Stewart is thirsty. Try beet juice or Thai tea.'
          else 'Stewart is hungry. Try Galbi, fish, or steak.'
        end
      );
    end if;

    v_next_craving := case v_game.food_craving
      when 'sweet' then 'thirsty'
      when 'thirsty' then 'hungry'
      else 'sweet'
    end;
  end if;

  v_result := public.care_for_stewart(p_action);

  update public.stewart_game
  set emily_care_count = emily_care_count + case when p_player = 'Emily' then 1 else 0 end,
      evan_care_count = evan_care_count + case when p_player = 'Evan' then 1 else 0 end,
      food_craving = coalesce(v_next_craving, food_craving)
  where id = 1
  returning emily_care_count, evan_care_count
  into v_emily_count, v_evan_count;

  return v_result || jsonb_build_object(
    'accepted', true,
    'food_craving', (select food_craving from public.stewart_game where id = 1),
    'care_counts', jsonb_build_object(
      'emily_care_count', v_emily_count,
      'evan_care_count', v_evan_count
    )
  );
end;
$$;

revoke all on function public.get_stewart_state() from public, anon, authenticated;
revoke all on function public.care_for_stewart(text) from public, anon, authenticated;
revoke all on function public.care_for_stewart_by_player(text, text, text) from public, anon, authenticated;
grant execute on function public.get_stewart_state() to anon, authenticated;
grant execute on function public.care_for_stewart(text) to anon, authenticated;
grant execute on function public.care_for_stewart_by_player(text, text, text) to anon, authenticated;
```

## Add caregiver counts to an existing project

Since your shared game table already exists, run this migration in a **new SQL Editor
query**. It preserves current game state, starts both caregiver counters at zero, and
adds a function that records each successful care action under the selected name.

```sql
alter table public.stewart_game
  add column if not exists emily_care_count integer not null default 0 check (emily_care_count >= 0),
  add column if not exists evan_care_count integer not null default 0 check (evan_care_count >= 0);

create or replace function public.care_for_stewart_by_player(p_action text, p_player text)
returns jsonb
language plpgsql
security definer
set search_path = public, pg_temp
as $$
declare
  v_result jsonb;
  v_emily_count integer;
  v_evan_count integer;
begin
  if p_player is null or p_player not in ('Emily', 'Evan') then
    raise exception 'Unknown player';
  end if;

  v_result := public.care_for_stewart(p_action);

  update public.stewart_game
  set emily_care_count = emily_care_count + case when p_player = 'Emily' then 1 else 0 end,
      evan_care_count = evan_care_count + case when p_player = 'Evan' then 1 else 0 end
  where id = 1
  returning emily_care_count, evan_care_count
  into v_emily_count, v_evan_count;

  return v_result || jsonb_build_object(
    'care_counts', jsonb_build_object(
      'emily_care_count', v_emily_count,
      'evan_care_count', v_evan_count
    )
  );
end;
$$;

revoke all on function public.care_for_stewart(text) from public, anon, authenticated;
revoke all on function public.care_for_stewart_by_player(text, text) from public, anon, authenticated;
grant execute on function public.care_for_stewart_by_player(text, text) to anon, authenticated;
```

Reload the website after the migration. The Emily/Evan choice is a quick name selector,
not password-protected authentication; anyone using the page can choose either name.

## Add shared food cravings to an existing project

Run this migration after the caregiver-count migration. It keeps the existing needs and
care counts, initializes the shared craving to sweet, and replaces the caregiver RPC
with food validation. Accepted food follows the shared cycle sweet → thirsty → hungry;
refused food does not change Stewart or either caregiver count. The website sends only
the food key, while this function enforces which food is appropriate.

```sql
alter table public.stewart_game
  add column if not exists food_craving text not null default 'sweet'
    check (food_craving in ('sweet', 'thirsty', 'hungry'));

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
    'food_craving', v_game.food_craving,
    'goal_target', v_game.goal_target,
    'complete', v_game.is_complete
  );
end;
$$;

drop function if exists public.care_for_stewart_by_player(text, text);
create or replace function public.care_for_stewart_by_player(
  p_action text,
  p_player text,
  p_food text default null
)
returns jsonb
language plpgsql
security definer
set search_path = public, pg_temp
as $$
declare
  v_result jsonb;
  v_game public.stewart_game%rowtype;
  v_elapsed_hours numeric;
  v_fullness numeric;
  v_next_craving text;
  v_emily_count integer;
  v_evan_count integer;
begin
  if p_player is null or p_player not in ('Emily', 'Evan') then
    raise exception 'Unknown player';
  end if;

  if p_action = 'food' then
    select * into v_game from public.stewart_game where id = 1 for update;
    v_elapsed_hours := greatest(
      extract(epoch from (clock_timestamp() - v_game.updated_at)),
      0
    ) / 3600.0;
    v_fullness := round(greatest(0::numeric, v_game.fullness - 12 * v_elapsed_hours), 1);

    if round(v_fullness) >= 100 then
      return jsonb_build_object(
        'accepted', false,
        'food_craving', v_game.food_craving,
        'message', 'Stewart is too full to eat right now.'
      );
    end if;

    if p_food is null or p_food not in ('galbi', 'fish', 'steak', 'msr', 'thai_tea', 'roll', 'beet_juice', 'ice_cream') then
      raise exception 'Unknown food item';
    end if;

    if (v_game.food_craving = 'sweet' and p_food not in ('ice_cream', 'msr', 'roll', 'thai_tea'))
      or (v_game.food_craving = 'thirsty' and p_food not in ('thai_tea', 'beet_juice'))
      or (v_game.food_craving = 'hungry' and p_food not in ('galbi', 'fish', 'steak')) then
      return jsonb_build_object(
        'accepted', false,
        'food_craving', v_game.food_craving,
        'message', case v_game.food_craving
          when 'sweet' then 'Stewart wants something sweet. Try ice cream, mango sticky rice, a cinnamon roll, or Thai tea.'
          when 'thirsty' then 'Stewart is thirsty. Try beet juice or Thai tea.'
          else 'Stewart is hungry. Try Galbi, fish, or steak.'
        end
      );
    end if;

    v_next_craving := case v_game.food_craving
      when 'sweet' then 'thirsty'
      when 'thirsty' then 'hungry'
      else 'sweet'
    end;
  end if;

  v_result := public.care_for_stewart(p_action);

  update public.stewart_game
  set emily_care_count = emily_care_count + case when p_player = 'Emily' then 1 else 0 end,
      evan_care_count = evan_care_count + case when p_player = 'Evan' then 1 else 0 end,
      food_craving = coalesce(v_next_craving, food_craving)
  where id = 1
  returning emily_care_count, evan_care_count
  into v_emily_count, v_evan_count;

  return v_result || jsonb_build_object(
    'accepted', true,
    'food_craving', (select food_craving from public.stewart_game where id = 1),
    'care_counts', jsonb_build_object(
      'emily_care_count', v_emily_count,
      'evan_care_count', v_evan_count
    )
  );
end;
$$;

revoke all on function public.get_stewart_state() from public, anon, authenticated;
revoke all on function public.care_for_stewart_by_player(text, text, text) from public, anon, authenticated;
grant execute on function public.get_stewart_state() to anon, authenticated;
grant execute on function public.care_for_stewart_by_player(text, text, text) to anon, authenticated;
```

## Add shared cleanliness and dental care

Run this migration after the shared food cravings migration. Existing saves start at
100% cleanliness. Each accepted food serving reduces it by 55 points, making his teeth
look dirty; refused food does not affect cleanliness. Players can brush his teeth in the
bathroom to restore cleanliness to 100%, and brushing counts as a care action. If you
already ran this migration, run this same SQL again to update the food penalty and add
the extra-food purchase function. After all matching foods are used in a kitchen visit,
players can buy and serve one extra item from the current craving category for 10 points.

```sql
alter table public.stewart_game
  add column if not exists cleanliness numeric not null default 100
    check (cleanliness between 0 and 100);

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
      'fun', round(greatest(0::numeric, v_game.fun - 14 * v_elapsed_hours), 1),
      'cleanliness', round(v_game.cleanliness, 1)
    ),
    'level', v_game.level,
    'points', v_game.points,
    'turns', v_game.turns,
    'food_craving', v_game.food_craving,
    'goal_target', v_game.goal_target,
    'complete', v_game.is_complete
  );
end;
$$;

create or replace function public.care_for_stewart_by_player(
  p_action text,
  p_player text,
  p_food text default null
)
returns jsonb
language plpgsql
security definer
set search_path = public, pg_temp
as $$
declare
  v_result jsonb;
  v_game public.stewart_game%rowtype;
  v_elapsed_hours numeric;
  v_fullness numeric;
  v_next_craving text;
  v_cleanliness numeric;
  v_emily_count integer;
  v_evan_count integer;
begin
  if p_player is null or p_player not in ('Emily', 'Evan') then
    raise exception 'Unknown player';
  end if;

  if p_action = 'food' then
    select * into v_game from public.stewart_game where id = 1 for update;
    v_elapsed_hours := greatest(
      extract(epoch from (clock_timestamp() - v_game.updated_at)),
      0
    ) / 3600.0;
    v_fullness := round(greatest(0::numeric, v_game.fullness - 12 * v_elapsed_hours), 1);

    if round(v_fullness) >= 100 then
      return jsonb_build_object(
        'accepted', false,
        'food_craving', v_game.food_craving,
        'message', 'Stewart is too full to eat right now.'
      );
    end if;

    if p_food is null or p_food not in ('galbi', 'fish', 'steak', 'msr', 'thai_tea', 'roll', 'beet_juice', 'ice_cream') then
      raise exception 'Unknown food item';
    end if;

    if (v_game.food_craving = 'sweet' and p_food not in ('ice_cream', 'msr', 'roll', 'thai_tea'))
      or (v_game.food_craving = 'thirsty' and p_food not in ('thai_tea', 'beet_juice'))
      or (v_game.food_craving = 'hungry' and p_food not in ('galbi', 'fish', 'steak')) then
      return jsonb_build_object(
        'accepted', false,
        'food_craving', v_game.food_craving,
        'message', case v_game.food_craving
          when 'sweet' then 'Stewart wants something sweet. Try ice cream, mango sticky rice, a cinnamon roll, or Thai tea.'
          when 'thirsty' then 'Stewart is thirsty. Try beet juice or Thai tea.'
          else 'Stewart is hungry. Try Galbi, fish, or steak.'
        end
      );
    end if;

    v_next_craving := case v_game.food_craving
      when 'sweet' then 'thirsty'
      when 'thirsty' then 'hungry'
      else 'sweet'
    end;
  end if;

  v_result := public.care_for_stewart(p_action);

  update public.stewart_game
  set emily_care_count = emily_care_count + case when p_player = 'Emily' then 1 else 0 end,
      evan_care_count = evan_care_count + case when p_player = 'Evan' then 1 else 0 end,
      food_craving = coalesce(v_next_craving, food_craving),
      cleanliness = case when p_action = 'food' then greatest(0::numeric, cleanliness - 55) else cleanliness end
  where id = 1
  returning emily_care_count, evan_care_count, cleanliness
  into v_emily_count, v_evan_count, v_cleanliness;

  return v_result || jsonb_build_object(
    'needs', (v_result->'needs') || jsonb_build_object('cleanliness', round(v_cleanliness, 1)),
    'accepted', true,
    'food_craving', (select food_craving from public.stewart_game where id = 1),
    'care_counts', jsonb_build_object(
      'emily_care_count', v_emily_count,
      'evan_care_count', v_evan_count
    )
  );
end;
$$;

create or replace function public.clean_stewart_teeth_by_player(p_player text)
returns jsonb
language plpgsql
security definer
set search_path = public, pg_temp
as $$
declare
  v_game public.stewart_game%rowtype;
  v_state jsonb;
  v_emily_count integer;
  v_evan_count integer;
begin
  if p_player is null or p_player not in ('Emily', 'Evan') then
    raise exception 'Unknown player';
  end if;

  select * into v_game from public.stewart_game where id = 1 for update;
  if not found then
    raise exception 'Shared Stewart game is not initialized';
  end if;
  v_state := public.get_stewart_state();

  if v_game.cleanliness >= 50 then
    return v_state || jsonb_build_object(
      'accepted', false,
      'message', 'Stewart''s teeth are already clean.'
    );
  end if;

  update public.stewart_game
  set fullness = coalesce((v_state #>> '{needs,fullness}')::numeric, fullness),
      energy = coalesce((v_state #>> '{needs,energy}')::numeric, energy),
      fun = coalesce((v_state #>> '{needs,fun}')::numeric, fun),
      cleanliness = 100,
      points = points + 5,
      turns = turns + 1,
      emily_care_count = emily_care_count + case when p_player = 'Emily' then 1 else 0 end,
      evan_care_count = evan_care_count + case when p_player = 'Evan' then 1 else 0 end,
      updated_at = clock_timestamp()
  where id = 1
  returning emily_care_count, evan_care_count
  into v_emily_count, v_evan_count;

  return public.get_stewart_state() || jsonb_build_object(
    'accepted', true,
    'message', 'Sparkling! Stewart''s teeth are all clean. +5 happy points.',
    'care_counts', jsonb_build_object(
      'emily_care_count', v_emily_count,
      'evan_care_count', v_evan_count
    )
  );
end;
$$;

create or replace function public.buy_extra_food_by_player(p_player text, p_food text)
returns jsonb
language plpgsql
security definer
set search_path = public, pg_temp
as $$
declare
  v_points integer;
  v_result jsonb;
begin
  if p_player is null or p_player not in ('Emily', 'Evan') then
    raise exception 'Unknown player';
  end if;

  select points into v_points
  from public.stewart_game
  where id = 1
  for update;

  if not found then
    raise exception 'Shared Stewart game is not initialized';
  end if;

  if v_points < 10 then
    return jsonb_build_object(
      'accepted', false,
      'points', v_points,
      'message', 'An extra serving costs 10 happy points.'
    );
  end if;

  v_result := public.care_for_stewart_by_player('food', p_player, p_food);
  if coalesce((v_result->>'accepted')::boolean, false) is not true then
    return v_result || jsonb_build_object('points', v_points);
  end if;

  update public.stewart_game
  set points = points - 15
  where id = 1
  returning points into v_points;

  return v_result || jsonb_build_object(
    'points', v_points,
    'message', 'Extra serving purchased for 10 happy points. Stewart enjoyed it!'
  );
end;
$$;

revoke all on function public.get_stewart_state() from public, anon, authenticated;
revoke all on function public.care_for_stewart_by_player(text, text, text) from public, anon, authenticated;
revoke all on function public.clean_stewart_teeth_by_player(text) from public, anon, authenticated;
revoke all on function public.buy_extra_food_by_player(text, text) from public, anon, authenticated;
grant execute on function public.get_stewart_state() to anon, authenticated;
grant execute on function public.care_for_stewart_by_player(text, text, text) to anon, authenticated;
grant execute on function public.clean_stewart_teeth_by_player(text) to anon, authenticated;
grant execute on function public.buy_extra_food_by_player(text, text) to anon, authenticated;
```
