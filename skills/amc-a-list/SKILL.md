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

## Deep links (verified)

Showtimes, filtered by date and format — one page load, no clicking:

```
https://www.amctheatres.com/movie-theatres/<market>/<theatre>/showtimes?date=YYYY-MM-DD&premium-offering=<format>
```

| Theater | market/theatre |
|---|---|
| AMC Columbia 14 | `baltimore/amc-columbia-14` |
| AMC Tysons Corner 16 | `washington-d-c/amc-tysons-corner-16` |
| AMC Hoffman Center 22 | `washington-d-c/amc-hoffman-center-22` |

Formats: `dolbycinemaatamcprime` (Dolby Cinema), `imax` (IMAX). Omit `date`
for today.

Read results with `browser_evaluate` on `main.innerText` (slice from
"movies showing"). Showtime links are `/showtimes/<id>`; the seat map is
`/showtimes/<id>/seats`. Matinee times may sit next to "UP TO 15% OFF" labels.

### Reading the seat map

Seats are checkboxes inside the grid labeled "Seat Selection Map", with labels
like "AMC Signature Recliner F10" or "Occupied AMC Signature Recliner F9".
Row A is closest to the screen; seat numbers run right-to-left on screen
(A18 at the left). Compact view in one call:

```js
() => { const g = document.querySelector('[aria-label="Seat Selection Map"]');
  const rows = {}; for (const b of g.querySelectorAll('input[type=checkbox],[role=checkbox]')) {
    const l = b.getAttribute('aria-label') || ''; const m = l.match(/([A-Z])(\d+)$/); if (!m) continue;
    (rows[m[1]] ||= []).push(/Occupied/i.test(l) || b.disabled ? 'x' : /Wheelchair Space/i.test(l) ? 'w' : '.'); }
  return Object.entries(rows).map(([r, s]) => r + ' ' + s.join('')).join('\n'); }
```

To pick a seat, `browser_find` its label (e.g. "Recliner F10") for the ref,
then click it.

## Procedure

1. **Showtimes:** deep link per theater for the requested date and formats.
   Check all three theaters unless he named one.
2. **Present 2–3 options** in one message: movie, theater, format, time, and a
   proposed seat for each (from the seat map). Mention if a premium show is
   nearly full.
3. **Wait for his pick.** A reply like "the 7:15" or "2" is the yes for that
   option.
4. **Reserve** with A-List. A-List pricing needs him signed in: if the page
   shows "Sign In or Join" at checkout, stop and send the viewer link (see
   `home-browser`). If checkout shows any charge (fees, upcharge, a
   non-A-List showtime), stop and tell him before continuing.
5. **Confirm** what was booked: movie, theater, time, seat, confirmation number.
6. Offer to add it to his Google Calendar (only add on yes).
