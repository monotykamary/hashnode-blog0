# Burning out Dataviews in Obsidian with Templater

## Introduction

At [Dwarves Foundation](https://dwarves.foundation/), we manage community learning and note submissions through a git-shared [Obsidian](https://obsidian.md/) Vault, our note-taking app, published through Obsidian Publish to our [Brainery](https://brain.d.foundation/). We use Dataview to aggregate frontmatter and metadata from our Vault.

We create lots of local reports in Obsidian, but there are certain reports or templates we want to publish publically for everyone to see. For us, our problem is that Obsidian Publish doesn't have a function to create dynamic views from Dataview queries on the web for us to make [Map of Contents](https://brain.d.foundation/Map+of+Content) (MOCs).

## What is Dataview?

[Dataview](https://github.com/blacksmithgu/obsidian-dataview) is a “a high-performance data index and query language” plugin for Obsidian. It has both DQL (SQL-like interface) and a fluent interface (method chaining) in JavaScript to query frontmatter from Obsidian Markdown files.

````plaintext
```dataview
task from #projects/active
```
````

![Task List](https://github.com/blacksmithgu/obsidian-dataview/raw/master/docs/docs/assets/project-task.png align="left")

````plaintext
```dataviewjs
for (let group of dv.pages("#book").where(p => p["time-read"].year == 2021).groupBy(p => p.genre)) {
	dv.header(3, group.key);
	dv.table(["Name", "Time Read", "Rating"],
		group.rows
			.sort(k => k.rating, 'desc')
			.map(k => [k.file.link, k["time-read"], k.rating]))
}
```
````

![Books By Genre](https://github.com/blacksmithgu/obsidian-dataview/raw/master/docs/docs/assets/books-by-genre.png align="left")

Dataview aggregates frontmatter to create its index. [Frontmatter](https://help.obsidian.md/Advanced+topics/YAML+front+matter) is file-level metadata that is both human-readable and Obsidian-readable:

```yaml
---
key: value
key2: value2
key3: [one, two, three]
key4:
- four
- five
- six
---
```

This allows us to treat each markdown file as an exotic record that we can aggregate and index to a database that we can query.

## Burning out dataviews?

Since dataview uses a code fence to interface dynamic results, there are times when we want to capture the result and save it statically as a Markdown file. This is especially the case when certain dataview queries are time-sensitive and may show a different set of data after a certain period.

### Problem

[https://github.com/blacksmithgu/obsidian-dataview/issues/42](https://github.com/blacksmithgu/obsidian-dataview/issues/42)

This problem is pretty common when you need to snapshot data for daily notes, publish them statically to a website, or have **notes that need to be updated regularly as a static page**.

Our issue refers to the latter. There aren't many solutions to this available publically, so I thought, why not just make it myself?

![Fine I'll do it myself Meme Generator - Imgflip](https://i.imgflip.com/356zn0.png?a463968 align="left")

### Burning out Dataviews with Templater

The easiest way to burn out dataviews is with templater. Templater has a few functions that applies a dynamic template to a static file:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1671182320249/6me3pidSa.png align="center")

For our example, we'll use our [Brainery's Latest Notes](https://brain.d.foundation/Latest+Notes) page. Here we integrate a bit of JavaScript inside \`&lt;%\* ... %&gt; Templater tags along with dataview queries to scaffold our template.

```plaintext
This is where we keep track of our top 10 latest Brainery notes:

<%*
const dv = this.app.plugins.plugins["dataview"].api;
const te = await dv.queryMarkdown(`LIST FROM -"_templates" AND -"_reports" AND -"challenge" WHERE date != NULL SORT date DESC LIMIT 10`);
tR += te.value;
%>

## Top Contributors this month
---
<%*
const topDiscordNotes = dv.pages(`-"_templates" AND -"_reports" AND -"challenge"`)
	.where(p => !!p.file.frontmatter.discord_id)
	.where(p => !!p.file.frontmatter.date)
    .where(p => dv.date(p.file.frontmatter.date) !== null)
    .where(p => dv.date(p.file.frontmatter.date).month === dv.date('today').month)
    .sort(p => p.date, "desc")
	.groupBy(p => p.discord_id)

const topAuthoredNotes = dv.pages(`-"_templates" AND -"_reports" AND -"challenge"`)
	.where(p => !!p.file.frontmatter.author)
	.where(p => !!p.file.frontmatter.date)
    .where(p => dv.date(p.file.frontmatter.date) !== null)
    .where(p => dv.date(p.file.frontmatter.date).month === dv.date('today').month)
    .sort(p => p.date, "desc")
	.groupBy(p => p.author);

tR += "| authors | notes |\n"
tR += "| ------- | ----- |\n"

for (let group of topDiscordNotes) {
	tR += `| ${group.key} | `
	for (let row of group.rows) {
		tR += ` [[${row.file.name}]]<br>`
	}
	tR += "|\n"
}

for (let group of topAuthoredNotes) {
	tR += `| ${group.key} | `
	for (let row of group.rows) {
		tR += ` [[${row.file.name}]]<br>`
	}
	tR += "|\n"
}
%>


## Newest Contributors
---
<%*
const discordNotes = dv.pages(`-"_templates" AND -"_reports" AND -"challenge"`)
	.where(p => !!p.file.frontmatter.discord_id)
	.where(p => !!p.file.frontmatter.date)
    .where(p => dv.date(p.file.frontmatter.date) !== null)
    .sort(p => p.date, "desc")
	.groupBy(p => p.discord_id)
	.filter(p => p.rows.length <= 1);

for (let group of discordNotes) {
	tR += `- **${group.key}**: `
	for (let row of group.rows) {
		tR += `${row.file.link}\n`
	}
}
%>
---
<%*
const authoredNotes = dv.pages(`-"_templates" AND -"_reports" AND -"challenge"`)
	.where(p => !!p.file.frontmatter.author)
	.where(p => !!p.file.frontmatter.date)
    .where(p => dv.date(p.file.frontmatter.date) !== null)
    .sort(p => p.date, "desc")
	.groupBy(p => p.author)
	.filter(p => p.rows.length <= 1);

for (let group of authoredNotes) {
	tR += `- **${group.key}**: `
	for (let row of group.rows) {
		tR += `${row.file.link}\n`
	}
}
%>

*This page was last modified at <%* tR += new Date().toISOString();%>*.
```

Now that we have our template, how do we make sure it updates without requiring us to access the static file, wiping it, and reapplying the template above?

#### Refreshing Recurring Templates Flow

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1671183835450/f7UrqPpjW.png align="center")

Here is a pretty crude flow-chart diagram of the actions to how this works. Essentially, both our **template** and the **target static file** we want to apply should have the `recurringTemplate` and `recurringTemplateName` keys in their frontmatter. Our action to apply the template will first **delete** the file, **recreate** the file, **then apply the template statically to it**.

For our latest notes, the frontmatter is as follows:

```plaintext
---
recurringTemplate: true
recurringTemplateName: latest-notes
---
```

#### The script to automate all of this

The one true script to automate refreshing recurring template static files is a pretty short one-page JavaScript script also written in Templater:

```plaintext
<%*
const dv = this.app.plugins.plugins["dataview"].api;
const recurringTemplatesList = dv.pages(`"_templates"`)
	.where(e => e.file.frontmatter.recurringTemplate);
const mappedRecurringTemplateNames = recurringTemplatesList.array().reduce((a, c) => {
	a[c.recurringTemplateName] = c.file.name
	return a;
}, {})

const matchingNotes = dv.pages(`!"_templates"`)
	.where(e => e.file.frontmatter.recurringTemplate);

for (const element of matchingNotes) {
	const { recurringTemplateName } = element
	const filePath = app.vault.getAbstractFileByPath(element.file.path);
	const folder = app.vault.getAbstractFileByPath(element.file.folder)

	// find the template
	const templatePath = mappedRecurringTemplateNames[recurringTemplateName];
	const template = tp.file.find_tfile(templatePath);

	// delete the file
	await app.vault.trash(filePath, true);

	// create a new file with the matching template
	await tp.file.create_new(template, element.file.name, false, folder);
}
%>
```

We'll name it as `Run - Update all recurring templates`. Opening our Templater modal will show us our script as a runnable template:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1671184150503/CvhxqN57r.png align="center")

This won't update any existing file you're on, and will run as a side effect to update all files that have the aforementioned frontmatter. Once run, we can see the updated pages in our `git status`:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1671184268749/ZhSQR53o7.png align="center")

## Closing thoughts

We've simplified automating recurring files, such as MOCs and our Latest Notes into a pretty clean interface. We rely on a script to help automate our logic, with the only manual requirement from the user is to add the 2 frontmatter keys as an indirect relation for update.

I hope this helps anyone also having a similar problem and just wants a script and some instructions to help manage this kind of workflow.