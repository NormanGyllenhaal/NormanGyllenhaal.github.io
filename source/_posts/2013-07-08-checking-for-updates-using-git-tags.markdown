---
layout: post
title: "Checking for updates using git tags"
date: 2013-07-08 19:39
comments: true
categories: [github, git-tag, curl, json-glib, automatic-updates]
---


At some point during the development of version 2.0 of my software package [XMI-MSIM](http://github.com/tschoonj/xmimsim), I decided to implement a routine that would allow for the program to check if newer versions (updates) were available. This would be extremely useful for the users of the Windows and Mac OS X builds of my package, since these operating systems do not come with package-management tools as most Linux distributions do (I am staying far away from Mac App Store and Windows store as I am not willing to pay their developer fees).

Initially I was looking at [Sparkle](http://sparkle.andymatuschak.org), a fantastic tool for OS X apps.
However, this would mean a different solution for my Windows build...

Since my goal is to keep the platform specific code as low as possible (#ifdef's really are quite ugly things), obviously I had to come up with a different solution.

<!--more-->

## Getting the tags with curl

In a rare moment of clarity, I came up with the idea of using git tags for this. Like most people, I am using the tags to indicate releases, and as a rule I include the version number in the tagname (e.g. XMI-MSIM-1.0). My method consists of having a routine called `check_for_updates` (what's in a name?), download the list of tags from github.com (using the [github v3 API for tags](http://developer.github.com/v3/git/tags/)). As I was writing (this part of) my application XMI-MSIM in C, I was looking for a library that could easily accomplish this. The quest yielded [libcurl](http://curl.haxx.se), an extremely versatile tool for transferring data using many, many protocols. The code I used for this was something like (full code at the end of this post):

{% codeblock lang:c %}

	//this line is obviously specific to your github account name and project name.
	//Change it accordingly
	#define XMIMSIM_GITHUB_TAGS_LOCATION "https://api.github.com/repos/tschoonj/xmimsim/git/refs/tags"

        char curlerrors[CURL_ERROR_SIZE];
 
 
        CURL *curl;
        CURLcode res;
        struct MemoryStruct chunk;
        
        chunk.memory = malloc(1);
        chunk.size = 0;
 
        fprintf(stdout,"checking for updates...\n");
        
        //setup curl
        curl = curl_easy_init();
        if (!curl) {
                fprintf(stderr,"Could not initialize cURL\n");
                return XMIMSIM_UPDATES_ERROR;
        } 
 
        curl_easy_setopt(curl, CURLOPT_URL,XMIMSIM_GITHUB_TAGS_LOCATION);
        curl_easy_setopt(curl, CURLOPT_SSL_VERIFYPEER, 0L);
        curl_easy_setopt(curl, CURLOPT_WRITEFUNCTION, WriteMemoryCallback);
        curl_easy_setopt(curl, CURLOPT_WRITEDATA, (void *)&chunk);
        curl_easy_setopt(curl, CURLOPT_USERAGENT, "libcurl-agent/1.0");
        curl_easy_setopt(curl, CURLOPT_ERRORBUFFER, curlerrors);
        curl_easy_setopt(curl, CURLOPT_CONNECTTIMEOUT, 4L);
        res = curl_easy_perform(curl);
        if (res != 0) {
                fprintf(stderr,"check_for_updates: %s\n",curlerrors);
                return XMIMSIM_UPDATES_ERROR;
        }
        curl_easy_cleanup(curl);
        
{% endcodeblock %}

## Parse the JSON code with Json-Glib and compare versions

The buffer that is returned in `chunk`, contains JSON code. To parse this, I used the Json-Glib library, a logical choice since my project is written in Gtk+ anyway... Code extract:

{% codeblock lang:c %}

  	parser = json_parser_new();
        if (json_parser_load_from_data(parser, chunk.memory, -1,&error) ==  FALSE) {
                if (error) {
                        fprintf(stderr,"check_for_updates: %s\n",error->message);
                        return XMIMSIM_UPDATES_ERROR;
                }
        }
        JsonNode *rootNode = json_parser_get_root(parser);
        if(json_node_get_node_type(rootNode) != JSON_NODE_ARRAY) {
                fprintf(stderr,"check_for_updates: rootNode is not an Array\n");
                return XMIMSIM_UPDATES_ERROR;
        }
        JsonArray *rootArray = json_node_get_array(rootNode);
        char *max_version = g_strdup(PACKAGE_VERSION);
        char *current_version = g_strdup(max_version);
        json_array_foreach_element(rootArray, (JsonArrayForeach) check_version_of_tag, &max_version);
 
        int rv;
        if (g_ascii_strtod(max_version, NULL) > g_ascii_strtod(current_version, NULL))
                rv = XMIMSIM_UPDATES_AVAILABLE;
        else
                rv = XMIMSIM_UPDATES_NONE;
 
        *max_version_rv = strdup(g_strstrip(max_version));
 
        g_object_unref(parser);
        fprintf(stdout,"done checking for updates\n");

{% endcodeblock %}

Important here is the `json_array_foreach_element` function, which will call `check_version_of_tag` for each tag, and update the highest found tag version number with each iteration.

After this, all that needs to be done is to compare this highest tag version with the internal version number (`PACKAGE_VERSION`, typically provided by a configure script), and the result is returned.

Now the method that I just described assumes that the version numbering is done with one major number and one minor number, allowing me to easily convert into a float for version comparison. Although this is sufficient for my personal needs, others may have to come up with a slightly more complex algorithm allowing to compare version numbers consisting of a major, minor and macro version number.

## The code

This is the full code: feel free to hack away at it.
It is taken from [xmimsim-gui-updater.c](https://github.com/tschoonj/xmimsim/blob/master/bin/xmimsim-gui-updater.c), which also contains code to download the new packages from a webserver (also using curl).

{% gist 5951294 %}
