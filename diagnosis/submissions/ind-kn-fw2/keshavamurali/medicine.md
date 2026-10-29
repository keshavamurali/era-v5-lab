# Medicine sheet: ind-kn-fw2 (keshavamurali)

Pack: `ind-kn-fw2` (Kannada, 500 documents). Counts come from `labels.csv` (500 labelled rows).

**Gold set used for the harm test:** the 118 documents of this pack that are labelled `clean` in `labels.csv`.
1% of 118 is 1.18 documents, so a medicine passes only if it changes or drops at most 1 of them.

**Built for a larger corpus too:** C2, C5 and G1 look at one document at a time. C1 counts lines per website
(the host of the `url` column) and stores only an 8-byte hash per line, so a common real line that many sites use
is never taken for a template. The runnable code is in `medicines.py`.

Shared helper used by the medicines below:

```python
import hashlib
import re
from collections import Counter
from urllib.parse import urlparse

END = re.compile(r'[.?!।॥"”\')]\s*$')   # a line that ends a sentence
```

## Medicine 1

1. **Disease:** C1 (boilerplate), 210 documents.
   - `5eaafa2efe204fb3` (row 8): `- Tag results for Subramanian Swamy`
   - `95fcd3a60a26a368` (row 19): `- Just In`, `- Don't Miss!`, `- News ಕರ್ನಾಟಕದಲ್ಲಿ 573 ಕೊರೊನಾ ಸೋಂಕಿತರು ಪತ್ತೆ`
   - `bfe4d0c9f43dc03b` (row 14): `‘ವಾರ್ತಾ ಭಾರತಿ’ ನಿಮಗೆ ಆಪ್ತವೇ ? …` and `ಬೆಂಬಲಿಸಲು ಇಲ್ಲಿ ಕ್ಲಿಕ್ ಮಾಡಿ` (the site's donation appeal, on all 7 of its pages)
2. **Medicine:** remove a line in any of these four cases:
   - (a) **site template:** the line is on 5 or more pages of the same website, and on at least 10% of that site's pages. A line that many different sites share, such as `ಧನ್ಯವಾದಗಳು` or a proverb, is never removed by this count. Documents without a URL are skipped by this part.
   - (b) it matches a short list of boilerplate phrases;
   - (c) it starts with a dash or bullet followed by an English word. Tag pages, related-news links and ads use this format: `- Tag results for …`, `- Education …`, `- News …`;
   - (d) it is a bare dash followed by a `<category> N days ago` line, a teaser card.
   ```python
   BOILER = re.compile(r'Share on WhatsApp|Copyright|©|Leave a Reply|Team Udayavani|ಟಾಪ್ ನ್ಯೂಸ್'
                       r'|\d+\s*(days?|hours?|minutes?) ago|Read more|ಇನ್ನಷ್ಟು (ಓದಿ|ಸುದ್ದಿ)', re.I)
   DASH_EN = re.compile(r'^[ \t]*[-–•][ \t]+[A-Za-z].*$', re.M)
   DASH_BARE = re.compile(r'^[ \t]*[-–•][ \t]*\n[^\n]*\d+\s*days? ago[^\n]*$', re.M)

   def host_of(url):
       h = urlparse(url or '').netloc.lower()
       return h[4:] if h.startswith('www.') else h

   def _key(line):
       return hashlib.blake2b(line.encode('utf-8'), digest_size=8).digest()

   def line_freq(docs):                       # docs: (host, text) pairs
       f, per_host = Counter(), Counter()
       for h, t in docs:
           if not h:
               continue
           per_host[h] += 1
           f.update((h, _key(l)) for l in {l.strip() for l in t.split('\n') if len(l.strip()) >= 3})
       return f, per_host

   def c1(t, host, freq, min_docs=5, min_share=0.1):
       f, per_host = freq
       t = DASH_BARE.sub('', t)
       t = DASH_EN.sub('', t)
       need = max(min_docs, min_share * per_host[host]) if host else float('inf')

       def template(l):
           s = l.strip()
           return len(s) >= 3 and f[(host, _key(s))] >= need
       return '\n'.join(l for l in t.split('\n') if not (template(l) or BOILER.search(l)))
   ```
3. **Action:** fix the text (remove the lines).
4. **Harm test:** 0 gold documents changed, 0 dropped. **Pass.**
   - The medicine changes 112 of the 210 C1 documents and 117 documents in the whole pack.
   - The dash rule (c, d) alone touches 53 documents and no gold document.
   - Part (a) finds 15 template lines in this pack, the same 15 as a whole-pack count. Each comes from 1 or 2 websites
     and covers 15% to 100% of that site's pages, for example `Don't Miss!` on all 34 `kannada.filmibeat.com` pages and
     `Source link` on all 8 `savikannada.in` pages.
5. **Side effects:**
   - A real Kannada bullet list that starts with an English word, such as `- WhatsApp ಮೂಲಕ…`, loses those lines.
   - A sentence that one site puts in many of its articles on purpose, such as a standard disclaimer, is removed.
   - Template lines on a site with fewer than 5 pages in the corpus are not found. A small pack finds less boilerplate than a large corpus.
   - Template lines that change from day to day, such as a "trending" headline box, can stay under the 10% share on a large site.
   - It catches only about half of the C1 documents. Menus and footers that are not repeated across a site's pages remain.

## Medicine 2

1. **Disease:** C5 (lists of keywords or links, teaser pages), 79 documents.
   - `dbbd14b33cfb0729` (row 37): `Display85`, `CPU43`, `GPU28` (spec widget and shop links)
   - `c472a3db00395ba6` (row 33): `ದಿನದ ಸುದ್ದಿ5 days ago` (related-headline teasers)
   - `bf21eb9f2a9511b6` (row 58): `Title`, `ಚಿತ್ರದುರ್ಗ | ನೀರಾವರಿ ಸಮಿತಿ ಅಧ್ಯಕ್ಷರ ಆಯ್ಕೆ ಪ್ರಕ್ರಿಯೆ ಸರಿಯಿಲ್ಲ : …` (headline list)
2. **Medicine:** short lines on their own are not a disease; poems, news highlights and credit blocks are short too.
   - **Block:** 3 or more consecutive short lines (under 60 characters, no sentence end), of which at least 2 carry a link or teaser signal: `N days ago`, `ಇನ್ನಷ್ಟು`, `Read more`, `Tag results`, a URL or `.com`, or `|`.
   - **Drop:** the whole document is dropped when more than 60% of its lines are short and at least 30% carry a signal.
   ```python
   SIGNAL = re.compile(r'\d+\s*(days?|hours?|minutes?|mins?) ago|ಇನ್ನಷ್ಟು|Read more|Tag results'
                       r'|https?://|www\.|\.com\b|\|', re.I)

   def _short(l):
       return 0 < len(l) < 60 and not END.search(l)

   def c5(t):
       lines = t.split('\n')
       idx = [i for i, l in enumerate(lines) if l.strip()]      # non-empty lines, original positions
       ls = [lines[i].strip() for i in idx]
       sig = [bool(SIGNAL.search(l)) for l in ls]
       if len(ls) >= 5 and sum(map(_short, ls)) / len(ls) > 0.6 and sum(sig) / len(ls) >= 0.3:
           return 'drop', ''
       drop, run = set(), []
       for k in range(len(ls) + 1):
           if k < len(ls) and (_short(ls[k]) or sig[k]):
               run.append(k)
           else:
               if len(run) >= 3 and sum(sig[j] for j in run) >= 2:
                   drop.update(idx[j] for j in run)
               run = []
       if not drop:
           return None, t
       return 'fix', '\n'.join(l for i, l in enumerate(lines) if i not in drop)
   ```
3. **Action:** fix the text (remove the block); drop the document when it is mostly a list.
4. **Harm test:** 0 gold documents changed, 0 dropped. **Pass.**
   - On its own, the medicine changes 8 of the 79 C5 documents.
   - Together with the dash rule of Medicine 1, 56 of the 79 C5 documents are treated.
   - An earlier version that used short lines alone changed 3 gold documents and failed:
     - `54be2713642d1bc6` (row 9), a vachana poem;
     - `73055b8f6994406f` (row 155), short news highlights;
     - `2a971489e8068e9c` (row 373), a folk-tale credit block.
   - Counting a leading `-` as a signal changed `03c55a28ac95ebbb` (row 151), a list of demands, so the dash was removed from the signals.
5. **Side effects:**
   - `|` is a signal, and Kannada verse uses `|` and `||` as line ends. Song lines in `634f9d4743893171` (row 31), such as `ಯಾರೇನೆಂದರು ನಿನಗೆ ಸದ್ಗುರು ದೇವ ||`, are removed with the headline block next to them.
   - A real list of links, for example a reference list at the end of an article, is removed.
   - Most C5 pages without these signals are not caught.

## Medicine 3

1. **Disease:** C2 (repeated lines), 29 documents.
   - `17e5d6b10dc06231` (row 111): the dateline and lead `ಕಲಬುರಗಿ,ಏ.5:ಈಶಾನ್ಯ ಕರ್ನಾಟಕದ ಪ್ರವಾಸೋದ್ಯಮದ ಅಭಿವೃದ್ಧಿಗೆ …` (188 characters) twice, as the title and again as the first line of the body
   - `76fdbcb28c49fc27` (row 51): `ಸುದ್ದಿದಿನ.ಕಾಂ|ವಾಟ್ಸಾಪ್|<phone>` four times (site contact line; number not copied)
   - `0bc105464a1dbdf3` (row 369): `ಫೋಟೋ: ಪೋಲಿನಾ ಇನೊಜೆಮ್ಟ್ಸೆವಾ, ಚಾಂಪಿಯನ್ಶಿಪ್` eight times
2. **Medicine:** if a line of **30 or more characters** appears **2 or more times** in one document, keep only its
   first copy.
   - Short lines are never touched, so repeated words, answers (`ಹೌದು`), dialogue tags and quiz options stay.
   - Documents that look like verse are skipped entirely: 6 or more lines, more than half of them short (under 60 characters) and without a sentence end. Refrains are the writing style there, and a refrain can be longer than 30 characters (`ಇವನಾರವ ಇವನಾರವ ಇವನಾರವನೆಂದೆನಿಸದಿರಯ್ಯಾ` is 40).
   ```python
   def _verse(t):
       ls = [l.strip() for l in t.split('\n') if l.strip()]
       return len(ls) >= 6 and sum(map(_short, ls)) / len(ls) > 0.5      # _short: see Medicine 2

   def c2(t, min_chars=30, times=2):
       if _verse(t):
           return t
       c = Counter(l.strip() for l in t.split('\n') if len(l.strip()) >= min_chars)
       rep = {l for l, n in c.items() if n >= times}
       seen, out = set(), []
       for l in t.split('\n'):
           s = l.strip()
           if s in rep:
               if s in seen:
                   continue
               seen.add(s)
           out.append(l)
       return '\n'.join(out)
   ```
3. **Action:** fix the text.
4. **Harm test:** 0 gold documents changed, 0 dropped. **Pass.**
   - The medicine changes 19 of the 29 C2 documents and 21 documents in the whole pack.
   - The other 2 have real repeats but no C2 label yet: `436fd1e4f828cb2a` (row 100), a 167-character paragraph
     twice, and `aa2a3e5196ee932e` (row 365), a 42-character line twice.
   - The first version (any line of 3 or more characters, 3 or more copies) changed only 7 of the 29 C2 documents.
     It missed the title-repeated-in-body pattern, and it could thin short repeated words in a larger corpus.
5. **Side effects:**
   - Short repeated junk is no longer removed by C2: `ಖರೀದಿಸಿ.` six times (row 369) and
     `ಚಿತ್ರಕೃಪೆ: rajkumar1220` five times (row 476). Medicine 1 catches such lines only when they repeat across a site's pages.
   - A long line that is repeated on purpose in prose, such as a quoted sentence or a question restated in an FAQ, loses its second copy.
   - Verse with long lines (6 or more lines that are mostly 60 characters or longer) is not seen as verse, so a long refrain there is thinned.
   - The 30-character limit counts code points, which suits Kannada (about 3 to 4 words). It is a parameter, to be set again for other scripts.

## Medicine 4

1. **Disease:** G1 (word cut at the vowel sign), 28 documents.
   - Only 2 documents in the pack have the exact pattern that this medicine fixes, so there are 2 examples:
     - `634f9d4743893171` (row 31): `ವಣ ್ಸುವಾಗ` (space before the virama)
     - `2914f93d5aa778b8` (row 253): `ವಸಡಿನ ೊಂದನ್ನು` (space before a vowel sign)
   - Most G1 damage in this pack is another kind:
     - a split between two full letters (`ಕರ್ನಾ ಟಕದ`, `e427affbdf302a39`, row 48);
     - legacy-font damage (`ಸಾಂಸ್ಕøತಿಕ`).
   - This regex cannot fix those.
2. **Medicine:** a Kannada dependent sign (vowel sign, virama, anusvara, visarga) can never start a word. Remove the space in front of it.
   ```python
   G1 = re.compile(r'([\u0C80-\u0CFF]) ([\u0CBE-\u0CCD\u0CD5\u0CD6\u0C82\u0C83])')

   def g1(t):
       return G1.sub(r'\1\2', t)
   ```
3. **Action:** fix the text.
4. **Harm test:** 0 gold documents changed, 0 dropped. **Pass.** The medicine changes 2 of the 28 G1 documents.
5. **Side effects:**
   - The join restores valid bytes but not lost letters. If a letter was dropped together with the space, the joined word is still wrong.
   - Text that shows a sign on its own on purpose, such as a grammar lesson that writes `ಾ`, is joined to the word before it.

## Medicines rejected

- **A6 (wrong or mixed script), 33 documents. Fails.**
  - **Rule:** flag Latin or legacy-font letters joined to a Kannada letter. A single `s` that ends a Kannada word is allowed, because it is valid dialect: `ಒಂದs`, `ತಾನs` in `92305d5a724c194a` (row 73).
  - **Failure:** it still marks the gold document `41932ab38a6fafb6` (row 172), where `ka.gov.inಗೆ` is a URL with a normal Kannada case suffix.
  - Scripts cannot be fixed without a font map, so this is at most a flag, not a medicine.
- **C3 (truncated document), 13 documents. Left out.**
  - **Rule:** "the last line has no sentence end."
  - **Failure:** it changed 7 gold documents (5.9%), even after skipping credit lines (`ಚಿತ್ರ ಕೃಪೆ:`, `ವರದಿ:`). The cases:
    - a closing `,` (row 1);
    - an author name (row 462);
    - a narrator credit block (row 373);
    - a poem (row 9);
    - complete sentences with no full stop (rows 45, 144, 289).
  - A regex cannot tell these from a real cut. C3 is labelled by hand only.
- **G2 (lost ZWJ or ZWNJ), 273 documents. No medicine.**
  - The pack has no ZWNJ at all, so this is a pack-wide loss from upstream cleaning.
  - Restoring it needs a word list, so it is reported upstream instead.
- **D3, D4, D5 (93, 34, 7 documents). No medicine.**
  - D3 is labelled, not judged.
  - D4 and D5 need a reader or a model, not a regex.
- **F3 (harmful content), 11 documents.**
  - A keyword rule (`dating`, `ಡೇಟಿಂಗ್`, `HGH`, `sex stor…`) touched 0 gold documents, but it caught only 4 of the 11.
  - It is kept as a flag for review, not as an automatic drop.
