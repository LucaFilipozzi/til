# verifying identity with rel-me

The [rel-me microformat][1] allows one to add hyperlinks from one site to
another -- perhaps reciprocally -- in order to verify identity.

[Simon Willson][2]'s TIL [Verifying your GitHub profile on Mastodon][3]
describes a clever means by which to overcome Mastodon / GitHub limitations
relating to identity verification by utilizing a GitHub-pages site as an
intermediate hop.

Turns out that the above intermediate website approach is not necessary.
The Mastodon software only checks for the existence of the rel-me link
once:
* add the mastodon rel-me link as github profile website
* add github to mastodon (verification occurs)
* restore github profile website

By leveraging this once-only-verification, I have managed to add both
[github](https://github.org/LucaFilipozzi) and [debian](https://nm.debian.org/person/lfilipoz/) links to my
[fosstodon](https://fosstodon.org/@LucaFilipozzi) profile.

---
Copyright (c) 2022 Luca Filipozzi

[1]: https://microformats.org/wiki/rel-me
[2]: https://fedi.simonwillison.net/@simon
[3]: https://til.simonwillison.net/mastodon/verifying-github-on-mastodon
