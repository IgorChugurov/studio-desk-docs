# Backend brief feedback — Studio admin

The owner decides canon. This file records where the implementation follows a later decision of the owner.

1. **Role → sections table.** The brief says the table lives in the code. On 2026-10-08 the owner decided it lives in the database: a migration fills it once, and there is no screen to edit it. `overview.md` is updated to say this. Finer rights («only my classes») stay for the later section topics; a row in the table means the section is allowed.
2. **Interface language storage.** A table `interface_language`, one row per e-mail. The owner is not a staff row, and the same e-mail shares one language across studios, so the language is not a column on `studio_staff`. No row means English. The server does not set the language cookie; the page writes that copy after the response.
