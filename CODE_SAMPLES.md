# Code samples

Excerpts from the private Grounded Docs repository (version 1.1.0), copied as they are, apart from a reference to an internal design record removed from one docstring; `# …` marks code left out between two parts of one file. They show the part of the system that matters most: an Answer may only say what the documents say.

## 1. Every Excerpt is checked word for word

The model returns claims, each citing a chunk number and an Excerpt. Before anything reaches a person, each Excerpt must appear in the chunk it cites. Comparison is on normalised text, so typographic quotes, dashes and spacing don't cause false failures, and an Excerpt under 20 characters proves nothing and is refused.

```python
# NFKC already turns the ellipsis into "..."; these quotes and dashes it leaves alone.
_TYPOGRAPHIC = str.maketrans(
    {
        **dict.fromkeys("\u2018\u2019\u201a\u201b\u2032", "'"),
        **dict.fromkeys("\u201c\u201d\u201e\u201f\u2033", '"'),
        **dict.fromkeys("\u2010\u2011\u2012\u2013\u2014\u2015\u2212", "-"),
    }
)
_WHITESPACE = re.compile(r"\s+")


def normalise(text: str) -> str:
    """SPEC normalisation: NFKC, typographic quotes and dashes to ASCII, whitespace collapsed."""
    text = unicodedata.normalize("NFKC", text).translate(_TYPOGRAPHIC)
    return _WHITESPACE.sub(" ", text).strip()
```

```python
MIN_EXCERPT_CHARS = 20  # counted after normalisation

# …


def _verify(draft_claim: DraftClaim, shown: list[SearchHit]) -> Claim | None:
    """The claim with only its verified Citations, or None if none verify.

    A Citation verifies when its `chunk_ref` names a shown chunk and its Excerpt, at least
    `MIN_EXCERPT_CHARS` long, appears in that chunk's text, after normalising both.
    """
    citations: list[Citation] = []
    for draft_citation in draft_claim.citations:
        if not 1 <= draft_citation.chunk_ref <= len(shown):
            continue
        hit = shown[draft_citation.chunk_ref - 1]
        excerpt = normalise(draft_citation.excerpt)
        # Too short to prove anything: "30 days" is a substring of half the corpus.
        if len(excerpt) < MIN_EXCERPT_CHARS or excerpt not in normalise(hit.text):
            continue
        citations.append(
            Citation(
                document_id=hit.document_id,
                document_title=hit.document_title,
                location=hit.location,
                excerpt=draft_citation.excerpt,
                source_link=hit.source_link,
            )
        )
    return Claim(text=draft_claim.text, citations=citations) if citations else None
```

## 2. The Outcome comes from the evidence, not from the model

Whatever the model says it found, the Outcome is decided by what survived verification. A model that claims an answer but whose every citation failed gets *Not Found*.

```python
def _outcome(draft: DraftAnswer, grounded: _Grounded, missing: list[str]) -> Outcome:
    """The Outcome the verified evidence supports, whatever the model claimed."""
    if draft.outcome == "not_found" or not (grounded.claims or grounded.conflicts):
        return "not_found"
    return "partially_answered" if missing else "answered"
```

A Conflict needs at least two verified sides from two different Documents; otherwise its verified claims are shown as ordinary claims.

```python
def is_conflict(claims: list[Claim]) -> bool:
    """At least two claims, citing at least two distinct Documents between them."""
    documents = {citation.document_id for claim in claims for citation in claim.citations}
    return len(claims) >= 2 and len(documents) >= 2
```

## 3. A hedged model call against tail latency

Most answers take about 3 seconds, but a provider sometimes stalls on one request. If the answer call hasn't returned after 5 seconds, an identical second call starts and the first success wins. At most two calls are in flight; the loser is abandoned. Which call won is reported to the request that made it, for its answer event. The `Runner` is a small port, so the tests drive this on a scripted timeline instead of real threads and sleeps.

```python
    def answer(self, standalone_question: str, chunks: list[ShownChunk]) -> DraftAnswer:
        def call() -> DraftAnswer:
            return self._inner.answer(standalone_question, chunks)

        started = self._runner.now()
        primary = self._runner.start(call)
        self._runner.wait([primary], self._hedge_after)
        if primary.done():
            return primary.result()  # an early failure is raised here, unhedged

        hedge = self._runner.start(call)
        calls: dict[Future[DraftAnswer], Winner] = {primary: "primary", hedge: "hedge"}
        first_failure: BaseException | None = None
        while calls:
            self._runner.wait(list(calls), None)
            for future in [f for f in calls if f.done()]:  # primary first: it wins a tie
                winner = calls.pop(future)
                failure = future.exception()
                if failure is None:
                    self._abandon(calls)
                    _report(winner)
                    self._log(winner, started)
                    return future.result()
                first_failure = first_failure or failure
        _report("none")
        self._log("none", started)
        assert first_failure is not None  # both calls finished, and neither succeeded
        raise first_failure
```

## 4. Tests pin the behaviour down

Integration tests run the real Assistant against a real Postgres with pgvector, with a scripted fake model. These three cover the rules above: a fabricated or out-of-range citation never reaches the person, an Answer with nothing verified is *Not Found*, and formatting differences don't cause false failures.

```python
def test_claims_whose_excerpt_is_not_in_a_shown_chunk_are_never_returned(index: Index) -> None:
    def respond(question: str, chunks: list[ShownChunk]) -> DraftAnswer:
        ref = _ref_of(chunks, PAYMENT_TERMS)
        return _draft(
            _claim("Invoices are due in 30 days.", ref, PAYMENT_TERMS),
            _claim("Invoices are due in 90 days.", ref, "Invoices are payable within 90 days."),
            _claim("Invoices are due in 30 days.", len(chunks) + 1, PAYMENT_TERMS),
        )

    assistant = Assistant(
        index=index,
        embedder=FakeEmbedder(),
        language_model=FakeLanguageModel(respond),
        relevance_threshold=OPEN_GATE,
    )

    answer = assistant.ask("When are invoices payable?", [])

    assert [c.text for c in answer.claims] == ["Invoices are due in 30 days."]
    assert answer.dropped_claim_count == 2
    assert all("90" not in c.excerpt for claim in answer.claims for c in claim.citations)


def test_an_answer_whose_every_excerpt_is_fabricated_is_not_found(index: Index) -> None:
    model = FakeLanguageModel(
        lambda question, chunks: _draft(
            _claim("Invoices are due in 90 days.", 1, "Invoices are payable within 90 days.")
        )
    )
    assistant = Assistant(
        index=index, embedder=FakeEmbedder(), language_model=model, relevance_threshold=OPEN_GATE
    )

    answer = assistant.ask("When are invoices payable?", [])

    assert answer.outcome == "not_found"
    assert answer.claims == []
    assert answer.dropped_claim_count == 1


def test_excerpts_match_despite_typographic_quotes_dashes_and_spacing(index: Index) -> None:
    plain = 'Approval requires the client\'s "written consent"  - email is enough.'
    model = FakeLanguageModel(
        lambda question, chunks: _draft(
            _claim("Consent must be written.", _ref_of(chunks, "written consent"), plain)
        )
    )
    assistant = Assistant(
        index=index, embedder=FakeEmbedder(), language_model=model, relevance_threshold=OPEN_GATE
    )

    answer = assistant.ask("What does approval require?", [])

    assert answer.outcome == "answered"
    assert answer.dropped_claim_count == 0
```

`_draft`, `_claim` and `_ref_of` are small test helpers: build the model's draft, build a claim citing a chunk, and find the number of the shown chunk containing a sentence.
