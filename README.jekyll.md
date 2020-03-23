# Migrating from Mixer to Jekyll

This branch contains some first steps towards converting GapWWW to Jekyll.

The plan is to do this incrementally, by always running first Mixer, then Jekyll,
until no *.mixer files are left and we can remove use of Mixer.
This works because Mixer produces *.html files which Jekyll just copies over.

For it to work, we also need to update the [Release checklist] to include a step
where Jekyll is run. In general, it's a good idea to take a look at the [Release checklist]
anyway, just so you know what happens right now in a release, as that heavily affects
how the website is updated, too.

See also [this HackMD](https://hackmd.io/EUtMx_2mRTaIYYlWSaVI6A) for further notes.

**Table of contents**

- [Converting a mixer file to Jekyll HTML](#converting-a-mixer-file-to-jekyll-html)
- [Converting from HTML to Markdown](#converting-from-html-to-markdown)
- [Dealing with non-generated content](#dealing-with-non-generated-content)
- [Dealing with generated content](#dealing-with-generated-content)
  * [Generated content in `Packages/`](#generated-content-in--packages--)
  * [Generated content in `Doc/Bib/`](#generated-content-in--doc-bib--)
  * [Generated content in `Releases/`](#generated-content-in--releases--)
  * [Other generated content?](#other-generated-content-)
- [Converting various Mixer commands to Jekyll](#converting-various-mixer-commands-to-jekyll)
  * [`<mixer func=...`](#--mixer-func--)
  * [`<mixer manual=...`](#--mixer-manual--)
  * [`<mixer parsevar=...`](#--mixer-parsevar--)
  * [`<mixer part=...`](#--mixer-part--)
  * [`<mixer person=...`](#--mixer-person--)
  * [`<mixer var=...`](#--mixer-var--)
- [Mixer `*.tmpl` versus Jekyll `_layouts`](#mixer---tmpl--versus-jekyll---layouts-)

<small><i><a href='http://ecotrust-canada.github.io/markdown-toc/'>Table of contents generated with markdown-toc</a></i></small>


## Converting a mixer file to Jekyll HTML

In order to convert a file `foo.mixer`, usually the following steps suffice:

1. Rename `foo.mixer` to `foo.html`
2. Replace the Mixer header with a Jekyll header; i.e., replace this:

        <?xml version="1.0" encoding="utf-8"?>

        <mixer template="gw.tmpl">
        <mixertitle>SOME TITLE</mixertitle>

   by this:

        ---
        title: SOME TITLE
        layout: default
        ---

   For other mixer templates, you may need a different Jekyll layout
3. Search for `<mixer` commands and convert them suitably (details are discussed later
   in this document)
4. TODO: anything else?


The script `etc/convert_mixer_to_jekyll.sh` takes care of the above steps
as well as some additional transformations.


## Converting from HTML to Markdown

For simplicity, it is recommended to first convert `.mixer` files to HTML, and
then test the result. But in many cases, it makes sense to then go on and
convert the HTML to Markdown. But this is a low priority, so I recommend to
first do other conversions.


## Dealing with non-generated content

Most files in the website is just handwritten static stuff, and can be
converted one at a time by hand. This is in contrasted to generated content,
discussed in the next section.

Here is an incomplete list of stuff which should be of the non-generated kind:

- `gap.mixer`
- `search.mixer`
- `Contacts/` ?
- `Datalib/` (watch out for the symlinks `prim.html`, `tom.html`, `trans.html`)
- `DevelopersPages/` -> can be removed ?!? (Ask Alex)
- `Faq/`
- `HostedGapPackages/` -> can be removed ?!? (Ask Alex)
- ...



## Dealing with generated content

Various parts of this website are themselves generated by scripts. These
should not be converted manually; instead, the generation mechanism should be
overhauled. This can range from minor tweaks that modify the generated output
to be Jekyll compatible; to a complete change on how that data is generated.

### Generated content in `Packages/`

The data in here is generated by scripts in the
<https://github.com/gap-system/gap-distribution> repository. That's in itself
problematic, because it means that modifying this website also forces one to
deal with the inner workings of the package distribution. So instead of
modifying these scripts, I suggest a complete overhaul on how we handle this.

Here is how I would tackle this: all the distribution scripts would do is to
copy all the `PackageInfo.g` scripts into some place, e.g. into the `Packages`
directory (either one will have to rename each file to have unique name; or
else, create a subdir for each package, and put its `PackageInfo.g` file in
there).

Then the task of turning these into HTML would be moved completely into the
GapWWW repository. For this, I would suggest to adapt the `gap-distribution`
code as well as <https://github.com/gap-system/GitHubPagesForGAP/blob/gh-pages/update.g>
to read all those `PackageInfo.g` files, and convert them into YAML files,
inside a [Jekyll collection](https://jekyllrb.com/docs/collections/). Then
a single Jekyll template can be used to turn these all into separate pages
for each package, as well as an overview page and more.


### Generated content in `Doc/Bib/`

TODO: probably the scripts generating this all should be adapted, in particular:
- `Doc/Bib/MSC2010.g`
- `Doc/Bib/bibsort.g`
- `Doc/Bib/convbib.g`
- `Doc/Bib/updatebib.g`

Talk to Alexander Konovalov for info about those.


### Generated content in `Releases/`

I suggest to use a [Jekyll collection](https://jekyllrb.com/docs/collections/)
for this; each release would then be a YAML file like `_releases/4.10.2.yml`, and
a single template would render that into the desired output.

Of course the scripts generating those pages should then be adjusted to
generated the YAML instead.


### Other generated content?

TODO: is there other generated content? please added it here!


## Converting various Mixer commands to Jekyll

For details on these commands, please consult the Mixer manual
(i.e., <https://github.com/gap-system/Mixer/blob/master/mixer.tex>, which you can
`pdflatex`).


### `<mixer func=...`

These are calls to arbitrary Python code. Luckily, only two seem to be used in GapWWW,
and only in the templates, so only people working on the templates / layouts should
have to work on these. For details see the section on templates and layouts.


### `<mixer manual=...`

Note that `etc/convert_mixer_to_jekyll.sh` takes care of these, so the following is
mostly to help explain how that conversion works.

With Mixer, links to the GAP manual and also package manuals are done like this:

    <mixer manual="LABELNAME">TEXT</mixer>

With Jekyll, you can replace this by the following

    {% include ref.html label="LABELNAME" text="TEXT" %}

For example, we convert this

    <mixer manual="Reference: The Programming Language">programming&nbsp;language</mixer>

to this:

    {% include ref.html manual="Reference: The Programming Language" text="programming&nbsp;language" %}

For these references to work, we must know the mapping from labels to URLs.
For this, the files `_includes/ref.html` and `data/help.yml`
are crucial. The former is static, but the later needs to be updated with
each release. Right now, `_data/help.yml` is generated by the file
`sync_mixer_to_jekyll.py`, which in turn reads `lib/AllLinksOfAllHelpSections.data`;
and *that* is generated by the file `dev/LinksOfAllHelpSections.g` in the main
GAP repository. For details, please refer to the [Release checklist].
But as a rough summary, that's how the current steps to updating the manual refs
would look like:

1. Extract the new GAP version somewhere
2. Go into that directory, compile GAP in it
3. Run `./gap dev/LinksOfAllHelpSections.g`
4. move the produced `AllLinksOfAllHelpSections.data` into the `lib` directory of `GapWWW`
5. Run `sync_mixer_to_jekyll.py` to further convert this to the YAML file `_data/help.yml`

In the future, we should modify `dev/LinksOfAllHelpSections.g` to directly
generate the YAML. And on the day we don't use Mixer anymore, we can remove
the code which generates `AllLinksOfAllHelpSections.data`.

I would also suggest that we move `dev/LinksOfAllHelpSections.g` from the main
GAP repository over to the GapWWW repository, as it seems to be strictly for
updating the website, and nothing else, so it makes more sense to co-develop it
with the rest of the website infrastructure


### `<mixer parsevar=...`

This is replaced by a full tree from the variable that has the name of the value of the attribute.

Usage of this is limited to a few files:
- `Datalib/datalib.mixer`: used to insert links to a few packages; easy to replace
- `Doc/manuals.mixer`: ?
- `Download/upgrade.mixer`: ?
- `Packages/packages.mixer`: used to generate a list of links to all packages;
  can be replaced by a Jekyll `for` loop


### `<mixer part=...`

Any element of type mixer having an attribute part is replaced by a full tree from another file.
(For details, refer to the Mixer manual).

Usage of this is limited to templates, generated files in `Packages/`, and `Doc/Bib/statistics.mixer`.
None should bother you for normal page conversions.


### `<mixer person=...`

Access to the person database. Some concrete examples (listing all formats actually used by GapWWW):

    <mixer person="Firstname Lastname" data="contact"/>
    <mixer person="Firstname Lastname" data="email_link"/>
    <mixer person="Firstname Lastname" data="name_city"/>
    <mixer person="Firstname Lastname" data="name_link"/>
    <mixer person="Firstname Lastname" data="name"/>

and, as a final special case (which we can just hardcode instead):

    <mixer person="GAP" data="address"/>

**TODO**: to support this, we should convert `lib/addresses` to a YAML file, say
`_data/addresses.yml` or `_data/persons.yml`, with the names as keys. Then
provide a bunch of files in `_includes` similar to how we deal with
`<mixer manual=...`


### `<mixer var=...`

Occurrences of `<mixer var="VARNAME"/>` usually can be replaced by
`{{ site.data.gap.VARNAME }}`, but exceptions may apply (as always:
test the result and verify it produces output that looks as the original
output did).

Some of these are already take care of by the `etc/convert_mixer_to_jekyll.sh` script.
The remaining ones are:

- `<mixer var="timestamp"/>`: we could either just remove them; or use a Jekyll plugin to emulate the effect
- ...


## Mixer `*.tmpl` versus Jekyll `_layouts`

TODO: list which tmp corresponds to which layout

TODO: deal with the navigation:


`<mixer func="maketop"/>` is used to insert the top level navigations. There
are plenty of ways to achieve that in Jekyll; in the worst case, we just hardcode something

`<mixer func="makeleft_plus"/>`: TODO: this implements the "Navigation Tree"
and might be a bit more annoying to replicate exactly. However, I think it is OK
for us to not implement a 100% match, as long as navigation is possible
(indeed, the current navigation system on the GAP website isn't exactly great,
so there is no point in replicating it faithfully anyway).



* [Release checklist]: https://github.com/gap-system/gap-distribution/blob/master/DistributionUpdate/RELEASE_CHECKLIST.md