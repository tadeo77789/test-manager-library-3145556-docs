# ADR-008 · Frontend known issues (not fixed)

- **Status:** accepted — known deviation, not fixed (13-oct-2026).
- **Context:** in-browser walkthrough of `app` in Chrome (T-22, 13-oct ~15:30) against the real stack. The flow
  login → lend → return → overdue → report works, and the error messages of CA-01.2, CA-06.5 and CA-10.2 are shown. Two issues:
  1. **Stale search results:** typing "cien" in the catalog showed 2 rows ("Cien años de soledad" and "El origen de las especies"),
     while `GET /api/v1/books?q=cien` returns 1. The second row matches the intermediate text "ci". `app/src/pages/Books.jsx`
     fires a request per keystroke and keeps whichever response arrives last, without debounce or discarding out-of-order responses.
  2. **Native confirmation dialog:** "Dar de baja" uses `window.confirm` (`app/src/pages/Books.jsx`). It works, but it is a blocking
     browser dialog, it cannot be styled, and it was not exercised in the automated walkthrough.
- **Decision:** keep the code for the delivery and record the issues. The API behaves as HU-04 says; the defect is only in the screen.
- **Discarded alternatives:** fixing them in the closing window (closing rule).
- **Consequences:** while typing, the catalog can briefly show results of a shorter query; the final result is correct once the
  last request resolves in order. Soft delete (CA-03.5) is still enforced by the API (`BOOK_HAS_OPEN_LOANS`), regardless of the dialog.
  Fix later: debounce (~250 ms, as `Lend.jsx` already does) plus ignoring responses of superseded queries; replace `window.confirm`
  with an in-page confirmation.
- **How to verify:** type "cien" quickly in the catalog and compare with `GET /api/v1/books?q=cien`;
  `grep -n "window.confirm" ../app/src/pages/Books.jsx`.
