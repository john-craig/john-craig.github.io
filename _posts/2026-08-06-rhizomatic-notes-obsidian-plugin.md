---
layout: post
title: "Rhizomatic Note Taking with Obsidian"
---
# Rhizomatic Note Taking

I got start with Obsidian a few years ago. It satisfied my basic requirements: a pleasant-looking Markdown editor with offline capabilities. I was also intrigued by the zettelkasten system as a method for organizing ideas and information, and Obsidian's intralink system seemed like an effective way of accomplishing that.

But, as I continued to use Obsidian to expand my knowledge base, I quickly encountered a problem: taxonomy paralysis.

I found that before I could commit my ideas and thoughts to a note, I first had to decide very carefully where the note ought to fit within the taxonomy of my existing knowledge base. 

If I encountered an issue with an SSH connection while I was working on a project to automate deployments, did that information belong in a dedicated note about secure shell, or in the note about the project? If I was setting up network-bound disk encryption, should that be categorized as "system administration" or "cyber security"? Should I track the physical hardware of my devices separately from the operating system and software configurations installed on them? Did I need to differentiate computers that I physically used, like my workstation and laptop, from computers that mostly acted as servers, like my raspberry pis?

At a certain point, I recalled a concept I had heard from a [video essay](https://www.youtube.com/watch?v=RQ2rJWwXilw) about the philosophical concept of "rhizomes". I certainly don't claim to be an expert in the subject, but a brief definition from [Wikipedia](https://en.wikipedia.org/wiki/Rhizome_(philosophy)) will serve well:

>  [A rhizome is] an assemblage that allows connections between any of its constituent elements, regardless of any predefined ordering, structure, or entry point.

This sounded like exactly what I needed. A system that allowed elements to added into it without any concern for a predefined structure. This would remove the cognitive load of worrying about how the information was to be categorized, and if it needed to be added to an existing place or not. Ideally, I would record the content immediately, and then the structure linking content together would emerge organically later.

## Themagraphs

Obsidian already has a very effective mechanism for linking together different notes: intralinks. These are inline references to other notes or tags that can be inserted quite seamlessly while writing. This mechanism is already very well-suited for creating a rhizomatic knowledge base, because you can simply write some of the keywords in a note as intralinks and this will automatically link them to a corresponding note for that keyword (or just create a "loose" intralink if no note exists).

My problem with intralinks as Obsidian implemented them was that the scope was too big. Ideally, if I were totally dispensing with categorization, I'd be able to just brain-dump everything into one long daily note and be done with it, tagging relevant intralinks as I went along. But, I can visit a lot of different topics over the course of the day, and not all of them are meaningfully associated with one another. Just because I happen to talk about the gas mileage of my truck on the same day that I'm debugging a Rust script doesn't mean there's a meaningful semantic connection. 

This problem lead me to the concept of a "themagraph": a paragraph or cluster of paragraphs within a single note pertaining to a particular topic or theme. The themagraph was, for me, the ideal scope for the base element of my rhizomatic note taking network. I could start a new themagraph when I knew I wanted to transition to another topic, or quickly go back to a previous themagraph when I knew I had more to add.

However, to properly implement this vision, I needed to extend Obsidian's base functionality. This led me down the path of creating my own dedicated plugin that I could use to experiment and implement my ideas-- we'll get back to that plugin a bit later.

## Rhizomatic Queries

Getting past taxonomy paralysis was great, but I did want to eventually go back and assess the emergent patterns and links between my themagraphs. In order to do this, I needed a way to view all of the themagraphs associated with a certain intralink or group of intralinks.

This lead me naturally to a query grammar based around Obsidian intralinks. There was no need to make it overly complex, so I reused typical boolean operators. To match themagraphs containing both `[[foo]]` and `[[bar]]`:

```
[[foo]] && [[bar]]
```

to match themagraphs that included either:

```
[[foo]] || [[bar]]
```

I did want to be able to make broader queries, however, so I added the meta-operator:

```
*[[foo]]
```

This operator would match any themagraphs that included links which were included in any themagraphs that included `[[foo]]`. Obviously, this kind of an operator can become huge, but it's useful for a big very of a very broad topic, like `[[programming]]`.

## Named Queries

While queries were incredibly useful for searching and exploring my rhizomatic network, I found myself also wanting a way to save the queries in a way that could be referenced more easily later. To solve this problem, I came up with the idea of "named queries": Obsidian intralinks which served as a direct alias for a certain rhizomatic query.

To avoid introducing an entirely new mechanism for storing named queries, I simply defined named queries as any themagraph which contains a text body like so:

```
[[my-query-name]] = <my-rhizomatic-query>
```

Here, `<my-rhizomatic-query>` can be any valid rhizomatic query. When a themagraph is created containing this psuedocode, and nothing else, it defines a new named query.

Named queries are automatically expanded inside of another rhizomatic query. For example, if `[[my-query-name]]` is assigned to the query `[[foo]] && [[bar]]`, then the following expansion occurs:

```
[[my-query-name]] || [[baz]]
```

becomes,

```
([[my-query-name]] || ([[foo]] && [[bar]])) || [[baz]]
```

This way a rhizomatic query can use either the named query as a convenient shorthand, or any of the actual links of the query, and themagraphs will still be correctly matched.

## Rhizomatic Templates

Finally, to take full advantage of rhizomatic note taking, we need a semi-permanent way to actually view the themagraphs of these queries. This takes me to rhizomatic templates. These are simple Markdown-based template files, inspired by the [Templater plugin](https://community.obsidian.md/plugins/templater-obsidian).

If a file is created in the vault that has both a rhizomatic query and a rhizomatic template set in the Obsidian properties of its frontmatter, it will be dynamically populated based on the template layout and the themagraphs returned by the query.

The interesting part is that a template can use subqueries to divide those results into different sections. The special `$query` value represents the query from the file's frontmatter, and a template expression can add another condition to it. For example, a project file might look like this:

```yaml
---
rhizoid-query: "[[my-project]]"
rhizomatic-template: project
---
```

The corresponding `templates/project.md` file could separate a definition from the project's other notes:

```markdown
# <% title() %>

## Definition
<% $query && [[definition]] %>

## Additional details
<% $query && ![[definition]] %>

## Tasks
<% tasks($query) %>
```

The first subquery, `$query && [[definition]]`, places only the themagraphs linked to both the project and `[[definition]]` under the Definition heading. The negated subquery, `$query && ![[definition]]`, places the remaining project themagraphs under Additional details. The task expression uses the complete project query and renders the matching tasks at the end. As the project grows, these sections update automatically from the intralinks, without requiring me to maintain a list of notes by hand.

The same idea works for a broader rollup. For example, I could use `[[programming]] || [[writing]]` as the query for a daily journal and give it a template like this:

```markdown
# <% title() %>

## Related themagraphs
<% $query %>

## Open tasks
<% tasks($query) %>
```

The query combines two topics, so the journal becomes a single view of the programming and writing themagraphs that I have accumulated. I could also add more selective sections by composing further subqueries, such as `$query && [[project]]` for project-related material or `$query && ![[definition]]` for notes that are not definitions. In this way, the query determines which themagraphs belong in the file, while the template determines how I want to arrange them.

## Installation

If the rhizomatic knowledge system described in this article is of interest to you, the [Rhizomatic Notes plugin](https://github.com/john-craig/rhizomatic-themagraphs-obsidian) is public, so it is possible to install the plugin directly from its source. This is a little more involved than installing a plugin from Obsidian's community-plugin browser, but it only takes a few steps:

1. Install [Node.js](https://nodejs.org/) and Git if they are not already installed.
2. Clone the repository and install its dependencies:

   ```bash
   git clone https://github.com/john-craig/rhizomatic-themagraphs-obsidian.git
   cd rhizomatic-themagraphs-obsidian
   npm install
   npm run build
   ```

3. Create a directory for the plugin inside the vault at `.obsidian/plugins/rhizomatic-notes/`.
4. Copy `main.js`, `manifest.json`, and `styles.css` from the repository into that directory.
5. Restart or reload Obsidian, open **Settings → Community plugins**, and enable **Rhizomatic Notes**.

The plugin uses `notes` and `projects` as its default template directories. Templates can be stored in a vault-level `templates/` directory and selected with the `rhizomatic-template` frontmatter property. If those directories do not fit the way a vault is organized, they can be changed in the plugin settings.
