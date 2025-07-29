---
title: Getting lazy with Claude and FastMCP
description: >-
  Using FastMCP and Claude Desktop to search Github users
date: 2025-07-29 06:00:00 +0100
categories: [General Engineering]
tags: [fastmcp,claude]
tok: true
---

---

A few days ago I wanted to start growing my network by connecting with developers that share the same sort of skills with me and are based somewhere nearby.

I am Python engineer and I am based in United Kingdom so programming language and location were the most important criteria in my search.

## Exit LinkedIn
My initial thought was to jump in LinkedIn but oh boy.. LinkedIn's search is horrible. The filters are all over the place and hardly ever work accurately.

In addition, the nature of LinkedIn's profiles makes it very hard to locate a person with a certain type of skills.

This is very problematic, especially with Python which is a very versatile language and is used by many people who are not necessarily Python developers.

Thus, a LinkedIn search could bring up Machine Learning engineers, SRE developers, Devops engineers, or even accountants that have simply listed Python in their skillset.

## Enter Github
I soon realised that the most trusted source would be Github itself.

Let's face it, we are developers and our skills are reflected in the code we produce. Naturally, we can advertise ourselves in Github much more efficiently than in LinkedIn or any other social media platform.

If you have multiple Python projects in your account, chances are you are actively developing in Python. So that should be good enough.

Github has a pretty advanced search that allows you to look for users and projects through their GraphQL API.

For example, pasting the following query in the Github search bar, will (in theory) return users from Berlin that have repositories written primarily in Python.

```
location:berlin language:python type:user
```

This is great, because even if it is not a very refined search, it can still give you very good results.

## Enter Claude
Brilliant, at this point I had established that Github search could be a very good source of information, but hang on..

It's July 2025. Going through a search bar? Clicking "next page" every other 10 or so results? Nah..

Given all the hype with AI Agents and MCPs I thought we could do better.

I have been using [Claude](https://claude.ai/) Desktop for the past few months and it has just started growing on me so my thought was that running a local MCP server and hooking it to Claude wouldn't be a hard task to do.

And indeed, it wasn't!

## Enter FastMCP
Ok, first of all a disclaimer.. I am not fan of the whole "Fast" hype when it comes to naming python packages.

It started with FastAPI and it was all right for a while, but it has now started spining out of control.

Nevertheless, [FastMCP](https://gofastmcp.com/getting-started/welcome) is at least fast when it comes to setting the framework up!

Create a project using [uv](https://github.com/astral-sh/uv), add your tools and boom! You have a great MCP server running locally on your machine. It takes literally two minutes.

## Enter Github GraphQL API
Speaking of tools, in my case I had to create a simple tool that connects to Github's GraphQL API and performs a query.

Pretty basic stuff. The code looks something like the snippet below and it is self explanatory:

```python
from dataclasses import dataclass
import os
from typing import List, Optional, Tuple

import requests

from mcp.server.fastmcp import FastMCP


# Create an MCP server
mcp = FastMCP("Github Users")

# Replace with your GitHub token
TOKEN = os.environ.get("GITHUB_TOKEN")
HEADERS = {"Authorization": f"Bearer {TOKEN}"}


@dataclass
class User:
    name: str
    user: str
    bio: Optional[str]


@mcp.tool()
def list_github_users(
    location: str,
    language: str,
    first: int,
    after: Optional[str] = None
) -> Tuple[List[User], str]:
    """Get github users"""

    query = """
    query SearchUsers($query: String!, $first: Int!, $after: String) {
        search(
            query: $query
            type: USER
            first: $first
            after: $after
        ) {
            nodes {
                ... on User {
                    login
                    name
                    bio
                    location
                    url
                    followers {
                        totalCount
                    }
                }
            }
            pageInfo {
                endCursor
            }
        }
    }
    """

    variables = {
        "query": f"location:{location} language:{language}",
        "first": first,
        "after": after,
    }

    response = requests.post(
        "https://api.github.com/graphql",
        json={"query": query, "variables": variables},
        headers=HEADERS,
        timeout=10
    )

    data = response.json()

    users = []
    after = None
    try:
        for user in data["data"]["search"]["nodes"]:
            users.append(
                User(user["name"], user["url"], user.get("bio"))
            )
            after = data["data"]["search"]["pageInfo"]["endCursor"]
    except Exception as exc:
        print(exc)

    return users, after


if __name__ == "__main__":
    mcp.run(transport="stdio")
```
One very important detail to notice is how we've used the `first` and `after` variables in the GraphQL query.

These variables can be used to essentially paginate our results!

For example, we can initially request the fist 10 results and the query will provide the information along with the Cursor of the last item. This way we can use the Cursor to request another set of results that come after.

## Time for action
Once I had the MCP server running and tested (a bit..) it was show time!

Hooking your local server to Claude Desktop is easy peasy lemon squeezy.

I won't go into the technical details since there are a lot of guides over the internet, but essentially, the `list_github_users` tool was visible in my Claude application.

![List Github Users Tool](/assets/img/illustrations/claude-fastmcp-tool.png){: width="972" height="589" }

My first query was the following:

`Show me the first Github user from Berlin that uses the Python programming language`

And the result was as expected:

![First Query](/assets/img/illustrations/claude-fastmcp-first-query.png){: width="972" height="589" }

Claude had successfully parsed my text and called the MCP tool with the correct arguments:

```json
{
    "first": 1,
    "language": "Python",
    "location": "Berlin",
}
```

My second query was even better:

`Show me the second user`

Bear in mind that I kind of cheated here a bit because I explicitely requested the "second" user.

However, Claude didn't flinch!

The result was again as expected:

![Second Query](/assets/img/illustrations/claude-fastmcp-second-query.png){: width="972" height="589" }

Claude had not only successfully parsed my text, but also realised that it had to carry on from the previous item.

We can see that because in the arguments passed to the MCP tool the `after` variable was used and was set to the value of the previous Cursor.

```json
{
    "after": "Y3Vyc29y0jF=",
    "first": 1,
    "language": "Python",
    "location": "Berlin",
}
```

## Moving Forward
Although this is a toy example, it does perform quite well and it integrates very nicely with Claude Desktop.

I would certainly like to explore Github's API further and potentially add some filters in the tool so that it returns only users that have more than X Python projects in their account, to make it more credible.

Finally, if you like to know how you can integrate [Cursor](https://cursor.com/en) with your local MCP server in a similar fashion, have a go at this great blog by Jamie Chang! [First Look At MCP](https://blog.changs.co.uk/first-look-at-mcp.html)
