---
title: "Own the Cockpit, Rent the Engine: Introduction to AI Self-Hosting"
date: 2026-09-11
layout: single
---

*"How I ran my own knowledge bases and LLM chat interface on Open WebUI, months before I owned a good GPU."*

This article is aimed at 6-months-ago me, and anyone else who thinks “AI is sort of neat but I like hosting things myself and do I really have to go out and get a Mac Studio or RTX 5090 to do this stuff I dunno that seems like a lot”.

You do not need an expensive machine/GPU to start self-hosting an AI stack. Hosting a frontend like Open WebUI already nets you some real gains over the stock Claude/ChatGPT/Gemini interface – and you’ll need one anyway to host your own LLM! For the rest of the article I will be calling Claude/ChatGPT/Gemini the Bigly models to save my own sanity\*.

What can you do with just a frontend? The Bigly models have really nice frontends already, with all sorts of nice features that occasionally disappear. For me, the primary appeal is that I have a digital library and I was getting annoyed constantly uploading different documents to a Bigly. This also solves issues of “where did it get that info from”. When I run a query with Open WebUI it tells me exactly what documents it used. I even have a persona that will only use documents in provided Knowledge Bases instead of anything ‘internal’ to the model.

Some other great reasons for hosting a frontend (oh no a bullet point list):

- **OWU supports personas that you can link to knowledge bases.**

  I have a “Neelix” persona that provides cooking advice and amusing commentary, based on the annoying cook from Star Trek Voyager. I have differing personas for creative writing, TTRPG editions (like a D&D 3.5 specific one), software engineering design, and the aforementioned “Truth” persona that only uses documents. And if I want Neelix to comment on D&D I very well can.

- **The only cap on your knowledge base is your hard drive**

  (vs Custom GPT capping out at 20 files, Claude Projects files at 30MB)

- **Prompt templates for lightweight personas**

  OWU’s “Prompts” building block lets you save prompt templates with variables, invoked with a slash command in any chat

- **Access any model per persona for cost and model-specific advantages.**

  Creative writing can use GPT 4.x, Programming can use 5.

- **Stability. Model APIs don’t change nearly as much as the consumer facing models in the Bigly frontends.**

For the future full self-hoster once you have the personas set up you can change the model under the hood with a click of a dropdown and save button, and clone and back up personas with ease. I personally have “Neelix” and “Neelix GPT” – the latter for if my model is down for whatever reason.

Hosting a frontend is effectively “Bigly Models for Power Users” with the option to move to full self-hosting later.

There are several AI frontends on the market: Open WebUI (selfhostable, enterprise-scalable, and what I use), AnythingLLM (which might be better for my purposes since it’s RAG focused), LibreChat, and Onyx (this one more for enterprise). I will be writing about Open WebUI because it is the only one I have set up.

## “Alright alright, I’m sold. How do I set this up?”

**Run a container! Like everything else these days, OWU comes in a container. [Their docs](https://docs.openwebui.com/) are where I'd go first.**

I personally run everything in rootless podman. It’s how I started running containers at my job and the minor extra level of security is a pleasant bonus to me.

For me, I set up my system with Ansible playbooks.\*\* I have a link to a generalized version of them [here](https://github.com/Chessler/personal-llm-frontend). This is not my finest DevOps work, but it gets the job done. At some point I’ll re-do it.

The most important and annoying part of using Open WebUI is the settings. I recommend using an environment variable file and loading that in instead of using the UI. In my own words, the two don’t sync. If you use the UI configuration it saves it to an internal database and in my experience sometimes randomly doesn’t keep on updates. If you lock it to an .env file it will definitely persist. The downside is you have to know that any config changes you make in the UI will be completely trashed on container reboot or have no effect at all. Searching, I found other issues, and I will let the Bigly model speak:

> “There's also a nastier variant if you're running Redis for multi-replica setups: PersistentConfig values get cached in Redis, and you have to manually delete the cached keys before a changed env var will actually take effect again.”
>
> [https://github.com/open-webui/open-webui/issues/20830](https://github.com/open-webui/open-webui/issues/20830)

So set `ENABLE_PERSISTENT_CONFIG=False` and keep a file. You will save yourself headaches.

Another gotcha: In Ye Olden Days (like a few months ago, AI moves quick!) you had to set `ENABLE_SIGNUP=True` on first boot even if you wanted it off so you could make an Admin account, then change it to False. It looks like now you can do `ENABLE_INITIAL_ADMIN_SIGNUP=True` which does the same thing [according to the docs](https://docs.openwebui.com/reference/env-configuration/#enable_initial_admin_signup).

## Now for other tweaks I wish I did starting out – take a look at my playbooks for more information. But all three of the “addons” you see I wish I had done initially.

Set up PGVector as your vector database and if you can connect a sizeable SSD, do so. This will dramatically speed up retrieval. This is where your indexed stuff will index. It is way faster than what OWU does out of the box and when you switch you have to reindex everything, so just do it now. PGVector is the only one officially supported by OWU and it’s great anyway.

Run a container with Tika. Tika is a document parser. We need this because OWU’s default parser sucks to the point even the [documentation recommends Tika/Docling](https://docs.openwebui.com/getting-started/advanced-topics/scaling/#step-6-fix-content-extraction--embeddings). Another “just do it now” because otherwise some things won’t parse, or they’ll parse weird. You will need to ensure it is connected with environment variables such as the ones available in my template repo. Here they are if you want them for yourself, you probably want to bookmark this page anyway: [https://docs.openwebui.com/reference/env-configuration/#tika_server_url](https://docs.openwebui.com/reference/env-configuration/#tika_server_url)

Set up an embedder – I highly recommend a Jina embedder. But not their reranker (more on that in the next section).  An embedder is a model that indexes your knowledge bases and retrieves them later. So if you set up another one later, you have to reindex all your knowledge bases with the new embedder. “Wait!” you may think. “You just said model there! I thought I didn’t have to host any models for this?” Well, yes, it Is running a model. But you can run an embedder on RAM if you want, it does not need a ton of performance nor VRAM. The model I use is [jina-embeddings-v5-text-small-retrieval](https://huggingface.co/jinaai/jina-embeddings-v5-text-small-retrieval), it runs on 640MB of VRAM and has quantizations down to 500MB. I put mine on my GTX because I have a shiny GTX. If you have a competent homelab setup computer you’ll be fine. You do not need to run out to get a 900090 or whatever NVIDIA will be on by the time you read this article.

Set `RAG_SYSTEM_CONTEXT=True`. I just found out this one while researching the article! By default RAG context gets injected into the user message, which shifts position every turn and defeats KV-cache/prompt-caching. The aforementioned setting pins it into the system message instead, which is a meaningful latency fix. Neato!

Do not bother with a reranker: I spent hours getting Jina Reranker working and when I did it didn’t do much for me. If you want one anyway, Quoth the Robot:

> “Reranker setup is rougher than it looks. People have hit literal boot-loops trying to use Jina reranker models in Open WebUI, where the container repeatedly tries and fails to load the model — worth a warning if you're steering readers toward Jina rerankers specifically, and worth noting RAG_RERANKING_MODEL_TRUST_REMOTE_CODE is the flag most people are missing.”

Here be dragons.

I would suggest starting with running an openwebui container, getting a little familiar with it, and then setting up PGVector/Tika/Embedder in one go. You are more than welcome to copy my playbook and tweak to your preferences.

To use OWU the only port you need to set a domain to is OWU’s, so in this case you would only need to reverse proxy 8090 or whatever you set the OWU port to. I use Caddy, it is crazy easy for a setup like this. I will not go into the further details of selfhosting here. If this is your first selfhosted service, I suggest setting up something like Jellyfin/Navidrome/Audiobookshelf/Calibre as well just to work out the kinks with your stack first.

Once you are running your OWU, give it some queries! Make a persona! I can’t give you much more than the OWU docs would do, and they would keep more up to date, so I would recommend perusing them.

I will write up a future article discussing why I chose the model I did and issues I encountered as I went through different versions. I am now on v3. For now, enjoy your shiny new frontend.

---

\* I know the common parlance is “Frontier” models, but I don’t consider them the frontier of AI tech. Bonsai models, KvarN, Dflash, Ornith, Qwen – THAT is the real frontier to me. Calling the Bigly models “the frontier” is like sitting in the 90s with your frosted tips declaring mainframes are the frontier of computing.

\*\* I always wonder how Le Guin would feel about the majority of the users of her coined term “ansible” largely being people who would never read her books. I only read Earthsea and The Left Hand of Darkness so I was even in that camp of having no idea “ansible” came from her. I did some digging (asked a Bigly model with search) and it couldn’t find any instance of her commenting on Ansible the software. The closest thing is commentary on trademarking Ansible: “Anyhow, the ansible is an Anarresti invention, and anarchists share stuff.”
