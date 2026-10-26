---
name: amc-a-list
description: Find AMC showtimes and book A-List reservations for Zach via his self-hosted "Browser" connector, using his theater, format, and seat preferences. Use for movie times, what's playing, or booking a movie.
---

# AMC A-List booking

Uses Zach's **Browser** connector and follows the `home-browser` skill's
consent rules (look freely; confirm before reserving; never sign in).

## Preferences

- **Format:** Dolby Cinema first — large formats are the reason to go instead
  of watching at home. Then IMAX. Standard only if he asks or nothing else exists.
- **Theaters (now, College Park):** check all three — AMC Columbia 14 (Dolby
  + IMAX with Laser, newer Dolby room, often the fastest drive), AMC Tysons
  Corner 16, and AMC Hoffman Center 22. A-List only works at AMC.
- **From August 2027 (Baltimore, Charles Village):** prefers walkable or inside
  the Beltway. For Dolby/IMAX on A-List, AMC White Marsh 16 is the nearest
  (outside the Beltway); Columbia 14 is the backup. Confirm with him once he's
  moved.
- **Seats:** not recorded yet. The first time, ask what he likes (row, center
  vs aisle) and suggest he add it here.
- **A-List allowance:** check remaining reservations on the account rather
  than assuming a number.

## Rules for AMC specifically

- AMC is behind Cloudflare bot protection. `fetch()` of AMC pages from a
  script gets 403. Only real page loads work, **one at a time** — never in
  parallel, never in a tight loop.
- Use the batched scripts below with `browser_run_code_unsafe`: one tool call
  does all the page loads. Keep each call under ~75 s total (the tunnel cuts
  requests off around 100 s); the scripts enforce this.

## Showtimes: all theaters, one call

Set `DATE` (`YYYY-MM-DD`, or `''` for today), `FORMAT`, and trim `THEATERS`
if he named one. Returns `{theater: [{title, times: ["7:00pm #<showtimeId>"]}]}`.

```js
async (page) => {
  const DATE = '2026-10-27', FORMAT = 'dolbycinemaatamcprime'; // or 'imax'
  const THEATERS = {
    'Columbia 14': 'baltimore/amc-columbia-14',
    'Tysons 16': 'washington-d-c/amc-tysons-corner-16',
    'Hoffman 22': 'washington-d-c/amc-hoffman-center-22',
  };
  const BUDGET = 75000, t0 = Date.now(), out = {};
  const p = await page.context().newPage();
  try {
    for (const [name, path] of Object.entries(THEATERS)) {
      if (Date.now() - t0 > BUDGET - 20000) { out[name] = 'skipped: time budget'; continue; }
      try {
        await p.goto(`https://www.amctheatres.com/movie-theatres/${path}/showtimes?${DATE ? 'date=' + DATE + '&' : ''}premium-offering=${FORMAT}`,
          { waitUntil: 'domcontentloaded', timeout: 25000 });
        await p.waitForSelector('a[href^="/showtimes/"], text=/No showtimes|No remaining|movies? showing/i', { timeout: 10000 }).catch(() => {});
        out[name] = await p.evaluate(() => {
          const res = [];
          for (const a of document.querySelectorAll('a[href^="/showtimes/"]')) {
            const time = (a.textContent.match(/\d{1,2}:\d{2}\s?[ap]m/i) || [])[0];
            if (!time) continue;
            let el = a.parentElement, title = '?';
            while (el && el !== document.body) {
              const ml = el.querySelector('a[href*="/movies/"]');
              if (ml && /\d+ HR/.test(el.textContent)) { title = ml.textContent.trim(); break; }
              el = el.parentElement;
            }
            let m = res.find(x => x.title === title); if (!m) res.push(m = { title, times: [] });
            m.times.push(`${time} #${a.getAttribute('href').split('/')[2]}`);
          }
          return res.length ? res : 'none';
        });
      } catch (e) { out[name] = 'error: ' + String(e.message).split('\n')[0]; }
    }
  } finally { await p.close(); }
  return { ms: Date.now() - t0, out };
}
```

Theater URL pattern if he asks about another AMC:
`/movie-theatres/<market>/<theatre-slug>/showtimes` (find the slug once via
the site search, then add it here).

## Seat maps: one or more showtimes, one call

Returns rows like `F ......xx.x........` (`.` open, `x` taken, `w`
wheelchair space). Row A is closest to the screen; seat numbers run
right-to-left on screen (A18 at the left, A1 at the right).

```js
async (page) => {
  const IDS = ['147563167']; // showtime ids from the showtimes call
  const out = {}, p = await page.context().newPage();
  try {
    for (const id of IDS) {
      try {
        await p.goto(`https://www.amctheatres.com/showtimes/${id}/seats`, { waitUntil: 'domcontentloaded', timeout: 25000 });
        await p.waitForSelector('[aria-label="Seat Selection Map"]', { timeout: 15000 });
        out[id] = await p.evaluate(() => {
          const g = document.querySelector('[aria-label="Seat Selection Map"]'), rows = {};
          for (const b of g.querySelectorAll('input[type=checkbox],[role=checkbox]')) {
            const l = b.getAttribute('aria-label') || ''; const m = l.match(/([A-Z])(\d+)$/); if (!m) continue;
            (rows[m[1]] ||= []).push(/Occupied/i.test(l) || b.disabled ? 'x' : /Wheelchair Space/i.test(l) ? 'w' : '.');
          }
          return Object.entries(rows).map(([r, s]) => r + ' ' + s.join('')).join('\n');
        });
      } catch (e) { out[id] = 'error: ' + String(e.message).split('\n')[0]; }
    }
  } finally { await p.close(); }
  return out;
}
```

## Procedure

1. **Showtimes:** one batched call for the date and format. Check all three
   theaters unless he named one.
2. **Seats:** one batched seat-map call for the 2–3 best candidates.
3. **Present 2–3 options** in one message: movie, theater, format, time, and a
   specific proposed seat for each. Mention if a premium show is nearly full.
4. **Wait for his pick.** A reply like "the 7:15" or "2" is the yes for that
   option.
5. **Reserve** (not yet tested end to end — verify and update this section on
   the first real booking): open `/showtimes/<id>/seats` in your own tab,
   `browser_find` the seat label (e.g. "Recliner F10"), click it, Continue, and
   proceed through tickets with the A-List option. A-List needs him signed in:
   on "Sign In or Join" at checkout, stop and send the viewer link (see
   `home-browser`). If checkout shows any charge, stop and tell him. Click the
   final confirm only after his yes for that exact showing and seat.
6. **Confirm** what was booked: movie, theater, time, seat, confirmation number.
7. Offer to add it to his Google Calendar (only add on yes).
