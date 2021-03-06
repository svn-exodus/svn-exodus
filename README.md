# svn-exodus

Tools to help extract one [Subversion®](https://subversion.apache.org/) repository from another.

* [svn-ancestry](https://github.com/svn-exodus/svn-ancestry) - A tool to trace the ancestry of a versioned file in a Subversion® repository and return a list of filenames that represent its various incarnations, across svn-copies, since its oldest ancestor was first svn-added.

# Business Case - The Problem

As a software developer, I have a large project whose code is stored in a [Subversion®](https://subversion.apache.org/) repository. Over the years I have refactored the code many times and teased it into a modular architecture made up of various components. One component has matured to the extent that it really ought to be a standalone project in itself. 

I want that component's code to reside in a new, standalone, Subversion® repository (to potentially be migrated into a `git` repository). But, I also want to retain the version history of all the files in that component so that I can read the log and the commit messages that explain how and why it came to be in its current form.



## Research Log

In order to understand my thought process (and therefore those of the potential users of the tool that I am making), I have noted the queries for some searches that I have done whilst trying to solve this problem. See the [Research Log](https://github.com/johnniewalker/svn-ancestry/wiki/ResearchLog) to get an idea of the context from which this tool was borne.

## Candidate Solution

Through my research, I have learned that one can follow the instructions, detailed under [Filtering Repository History in the Version Control with Subversion® Book](http://svnbook.red-bean.com/en/1.8/svn.reposadmin.maint.html#svn.reposadmin.maint.filtering), to hive off a part of a repository into a new one. 

In essence, you create a [dump of an existing repository](http://svnbook.red-bean.com/en/1.7/svn.ref.svnadmin.c.dump.html) and then use [`svndumpfilter`](http://svnbook.red-bean.com/en/1.7/svn.ref.svndumpfilter.html) to purge all but a choice set of commits from the dump file which can be imported into a new repository (using `svnadmin load`). 

As you stream data through `svndumpfilter`, you can either `include` or `exclude` certain paths (or patterns of paths) depending on situation. 

Before loading the commits into a new repository, one can optionally open up the resulting dump file in a text editor to do some string manipulation on the paths to strip out parent directories and hack them into shape.

So, we are 90% there towards our goal of extracting our component into its own Subversion® repository. But, the documentation states an issue in the penultimate paragraph of [Filtering Repository History](http://svnbook.red-bean.com/en/1.8/svn.reposadmin.maint.html#svn.reposadmin.maint.filtering):

> ...Also, copied paths can give you some trouble. Subversion supports copy operations in the repository, where a new path is created by copying some already existing path. It is possible that at some point in the lifetime of your repository, you might have copied a file or directory from some location that svndumpfilter is excluding, to a location that it is including. To make the dump data self-sufficient, svndumpfilter needs to still show the addition of the new path—including the contents of any files created by the copy—and not represent that addition as a copy from a source that won't exist in your filtered dump data stream. But because the Subversion repository dump format shows only what was changed in each revision, the contents of the copy source might not be readily available. If you suspect that you have any copies of this sort in your repository, you might want to rethink your set of included/excluded paths, perhaps **including the paths that served as sources of your troublesome copy operations, too**...

This means that, when we run the new project through the filter, in order to preserve their commit histories, we must not only `include` the project files at the current revision, but also `include` the paths of their ancestors. 

The question is: **How do we determine the paths of those ancestors?**

Also another issue is highlighted in the final paragraph of that [same section](http://svnbook.red-bean.com/en/1.8/svn.reposadmin.maint.html#svn.reposadmin.maint.filtering):

> ...Finally, svndumpfilter takes path filtering quite literally. If you are trying to copy the history of a project rooted at trunk/my-project and move it into a repository of its own, you would, of course, use the svndumpfilter include command to keep all the changes in and under trunk/my-project. But the resultant dump file makes no assumptions about the repository into which you plan to load this data. Specifically, the dump data might begin with the revision that added the trunk/my-project directory, but it will not contain directives that would create the trunk directory itself (because trunk doesn't match the include filter). You'll need to make sure that any directories that the new dump stream expects to exist actually do exist in the target repository before trying to load the stream into that repository.

So, in order to perform a successful [`svnadmin load`](http://svnbook.red-bean.com/en/1.7/svn.ref.svnadmin.c.load.html) of the selected commits, we must ensure that there is an appropriate directory structure for any svn-added files. For any file to be added, the parent directory must exist already or a svn add command must exist somewhere in the preceeding commit history.

The question is: **How do we determine the paths of those parent directories?**

This depends on the result of the previous question. It depends on the file's path and the paths of its ancestors.

# How do we determine the paths of a file's ancestors to use in the `include` arguments for `svndumpfilter`?

It is possible to run the [`svn log`](http://svnbook.red-bean.com/en/1.7/svn.ref.svn.c.log.html) command on a repository url. This would return its commit history. Using the [`--verbose` option](http://svnbook.red-bean.com/en/1.7/svn.ref.svn.html#svn.ref.svn.sw.verbose) ensures that we can see the paths affected by any given commit in that history.

We can ignore occassions when the file is modified. We are really interested in tracing the history up to the point when that file was svn-added to the repository. Furthermore, if that file was added using svn-copy (or any effective svn-move) command we want to trace the ancestry of that 'source file' too.

The information is all there in the output of `svn log`. 

In big repositories, with lots of history, it is probably a little time-consuming to trace this information manually. 

## So, how do we trace the ancestors *programmatically*?

The [`--xml` option](http://svnbook.red-bean.com/en/1.7/svn.ref.svn.html#svn.ref.svn.sw.xml) prints the `svn log` output in xml format. This makes it easier for a machine to understand it.

What we need is some tool to lift the pertinent ancestory path data from the xml stream that is output from a `svn log` command on a given repository file. If this is done for each file in the project-to-be-extracted then we can build a set of paths that need to be `included` in the `svndumpfilter` process that is run to filter the dump file.

# Does such a tool or solution already exist?

I don't know. I couldn't find it. If it doesn't exist I ought to make it.



