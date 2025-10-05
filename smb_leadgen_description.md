# Lead-Gen Pipeline — API I/O (Markdown)

> Все внешние вызовы опираются на официальные API провайдеров; внутри — наш тонкий оркестратор, нормализация, дедуп, сборка записи `company_record.v1`.

## 0) Итоговая модель (target)

**Сущность:** `company_record.v1`
**Поля (ядро):** `company{ id,name,website,domain,linkedin_url,industry,country_code,about,size_label,location,logo,extra }, embeddings{ model,vector[],text }, contacts[ { id,full_name,first_name,last_name,job_title,email,phone_number,location,seniority,departments[],linkedin_url,linkedin_profile_picture,linkedin_profile_description,skills[],experience[] } ]`. 

---
## 1) Google SERP (Bright Data)

**Назначение.** Получить до N результатов по запросу (компании/сайты) из Google с гео и языком, минуя капчи/парсинг.

**Офиц. доки:** Bright Data SERP API (обзор + Google endpoint). ([docs.brightdata.com][1])

**Наш эндпойнт (оркестратор).**
`POST /ingest/google-serp.search`

**Input**

```json
{
  "q": "site:linkedin.com/company fintech AND (\"VP of Engineering\" OR \"Head of Product\")",
  "country": "us",
  "hl": "en",
  "num": 100,
  "device": "desktop",
  "parse_mode": "json"
}
```

**Output**

```json
{
  "provider": "brightdata",
  "query": "…",
  "results": [
    {
      "title": "Acme Inc – LinkedIn",
      "url": "https://www.linkedin.com/company/acme",
      "snippet": "…",
      "rank": 1
    }
  ],
  "raw": { "provider_payload": { } }
}
```

---

## 2) LinkedIn Companies (Bright Data)

**Назначение.** Спарсить карточку компании (название, размер, индустрия, локации, сайт, about и т.п.).

**Офиц. доки:** LinkedIn Company Scraper / LinkedIn API scrapers. ([Bright Data][2])

**Наш эндпойнт.**
`POST /enrich/linkedin.company`

**Input**

```json
{
  "company_urls": ["https://www.linkedin.com/company/acme/"],
  "fields": ["name","industry","size","website","about","locations","logo"]
}
```

**Output**

```json
{
  "items": [
    {
      "linkedin_url": "https://www.linkedin.com/company/acme/",
      "name": "Acme Inc",
      "industry": "Computer Software",
      "size_label": "201-500",
      "website": "https://acme.com",
      "about": "…",
      "location": "San Francisco, CA",
      "logo": "https://media.licdn.com/…/logo.png",
      "extra": { "founded": 2017 }
    }
  ],
  "raw": { "provider_payload": { } }
}
```

---

## 3) Web-сайт компании → контент (Tavily Extract)

**Назначение.** Достать контент страниц (About, Careers, Contact, Pricing…) для извлечения сигналов и эмбеддингов.

**Офиц. доки:** Tavily Extract endpoint + best practices (raw_content, rate limiting и пр.). ([docs.tavily.com][3])

**Наш эндпойнт.**
`POST /fetch/extract`

**Input**

```json
{
  "urls": ["https://acme.com/about", "https://acme.com/contact"],
  "include_raw_content": true,
  "max_chars_per_page": 150000
}
```

**Output**

```json
{
  "pages": [
    {
      "url": "https://acme.com/about",
      "title": "About Acme",
      "raw_content": "Acme builds…",
      "links": ["mailto:sales@acme.com", "https://twitter.com/acme"],
      "status": 200
    }
  ],
  "raw": { "provider_payload": { } }
}
```

Python SDK example:
```python
from tavily import TavilyClient

# Step 1. Instantiating your TavilyClient
tavily_client = TavilyClient(api_key="tvly-YOUR_API_KEY")

# Step 2. Defining the list of URLs to extract content from
urls = [
    "https://en.wikipedia.org/wiki/Artificial_intelligence",
    "https://en.wikipedia.org/wiki/Machine_learning",
    "https://en.wikipedia.org/wiki/Data_science",
    "https://en.wikipedia.org/wiki/Quantum_computing",
    "https://en.wikipedia.org/wiki/Climate_change"
] # You can provide up to 20 URLs simultaneously

# Step 3. Executing the extract request
response = tavily_client.extract(urls=urls, include_images=True)

# Step 4. Printing the extracted raw content
for result in response["results"]:
    print(f"URL: {result['url']}")
    print(f"Raw Content: {result['raw_content']}")
    print(f"Images: {result['images']}\n")
```


---

## 4) LLM Cleaning / Normalization (DeepSeek)

**Назначение.** Привести разношёрстный веб-текст к аккуратным полям (about_short, product_list, geo, stack и т.п.), зачистить шум.

**Офиц. доки:** DeepSeek API (OpenAI-совместимый формат, базовый URL, примеры). ([api-docs.deepseek.com][4])

**Наш эндпойнт.**
`POST /clean/llm`

**Input**

```json
{
  "model": "deepseek-chat",
  "instructions": "Summarize company website into normalized JSON schema fields.",
  "schema": {
    "type": "object",
    "properties": {
      "about_short": {"type":"string"},
      "products": {"type":"array","items":{"type":"string"}},
      "hq_country": {"type":"string"},
      "tech_stack": {"type":"array","items":{"type":"string"}}
    },
    "required": ["about_short"]
  },
  "documents": [
    {"url":"https://acme.com/about","content":"…raw_content…"}
  ]
}
```

**Output**

```json
{
  "normalized": {
    "about_short": "Acme builds payment APIs…",
    "products": ["Payments API","Risk Scoring"],
    "hq_country": "US",
    "tech_stack": ["Python","PostgreSQL"]
  },
  "raw": { "provider_payload": { } }
}
```

> Примечание о тарификации в твоём плане (внутренний ориентир): `~$0.0045/сайт`. 

example: 
```python
from openai import OpenAI
import os

# Initialize the OpenAI client
client = OpenAI(
    # If you have not configured an environment variable, replace the following line with your Model Studio API key: api_key="sk-xxx"
    api_key=os.getenv("DASHSCOPE_API_KEY"),
    base_url="https://dashscope.aliyuncs.com/compatible-mode/v1",
)

messages = [{"role": "user", "content": "Who are you"}]
completion = client.chat.completions.create(
    # This example uses deepseek-v3.1. You can replace it with another model name, such as deepseek-v3 or deepseek-r1, as needed.
    model="deepseek-v3.1",
    messages=messages,
    # Set enable_thinking in extra_body to enable thinking mode. This parameter is valid only for deepseek-v3.1. Setting it for deepseek-v3 and deepseek-r1 does not cause an error.
    extra_body={"enable_thinking": True},
    stream=True,
    stream_options={
        "include_usage": True
    },
)

reasoning_content = ""  # Full thinking process
answer_content = ""  # Full response
is_answering = False  # Specifies whether the response phase has started
print("\n" + "=" * 20 + "Thinking process" + "=" * 20 + "\n")

for chunk in completion:
    if not chunk.choices:
        print("\n" + "=" * 20 + "Token usage" + "=" * 20 + "\n")
        print(chunk.usage)
        continue

    delta = chunk.choices[0].delta

    # Collect only the thinking content
    if hasattr(delta, "reasoning_content") and delta.reasoning_content is not None:
        if not is_answering:
            print(delta.reasoning_content, end="", flush=True)
        reasoning_content += delta.reasoning_content

    # After content is received, start to reply
    if hasattr(delta, "content") and delta.content:
        if not is_answering:
            print("\n" + "=" * 20 + "Full response" + "=" * 20 + "\n")
            is_answering = True
        print(delta.content, end="", flush=True)
        answer_content += delta.content
```

---
## 5) Parse employees by company name

**Описание**: Используем brightdata google search api, на основе нашего ICP ищем линкедины сотрудников, например
`site:linkedin.com/in ("CTO" OR "Head of Product") "at Stripe" -"Ex-" -"Former"`,
`site:linkedin.com/in "Stripe" ("present" OR "current")`


---
## 6) Parse linkedin profiles

**Description**: работает на базе brightdata. Нужен чтоб спарсить линкедины сотрудников, но без постов.

**Endpoint**: `POST https://api.brightdata.com/datasets/v3/trigger?dataset_id=gd_l1viktl72bvl7bjuj0&include_errors=true` 

**Input**: 

```json
[
{"url":"https://www.linkedin.com/in/elad-moshe-05a90413/"},
{"url":"https://www.linkedin.com/in/jonathan-myrvik-3baa01109"},
{"url":"https://www.linkedin.com/in/aviv-tal-75b81/"},
{"url":"https://www.linkedin.com/in/bulentakar/"}
]
```

**Output**:

```json
[
  {
    "id": "deb***dur***-a3*********",
    "name": "Debra D****d",
    "city": "Lincoln, Nebraska, United States",
    "country_code": "US",
    "position": "--",
    "about": null,
    "posts": null,
    "current_company": {
      "location": null
    },
    "experience": null,
    "url": "htt***//w***lin*********/in*********************07",
    "people_also_viewed": null,
    "educations_details": null,
    "education": null,
    "recommendations_count": null,
    "avatar": "https://static.licdn.com/aero-v1/sc/h/9c8pery4andzj6ohjkjp54ma2",
    "courses": null,
    "languages": null,
    "certifications": null,
    "recommendations": null,
    "volunteer_experience": null,
    "followers": null,
    "connections": null,
    "current_company_company_id": null,
    "current_company_name": null,
    "publications": null,
    "patents": null,
    "projects": null,
    "organizations": null,
    "location": "Lincoln",
    "input_url": "htt***//w***lin*********/in*********************07",
    "linkedin_id": "deb***dur***-a3*********",
    "activity": null,
    "linkedin_num_id": "881205451",
    "banner_image": "https://static.licdn.com/aero-v1/sc/h/5q92mjc5c51bjlwaj3rs9aa82",
    "honors_and_awards": null,
    "similar_profiles": [],
    "default_avatar": true,
    "memorialized_account": false,
    "bio_links": [],
    "first_name": "******",
    "last_name": "******"
  }
]
```
---
## 7) Parse employee posts

**Description**: Собрать посты сотрудника для обогащения.

**Endpoint**: `POST https://api.brightdata.com/datasets/v3/trigger?dataset_id=gd_lyy3tktm25m4avu764&include_errors=true&type=discover_new&discover_by=profile_url`

**Input**

```json
[{"url":"https://www.linkedin.com/in/bettywliu","start_date":"2018-04-25T00:00:00.000Z","end_date":"2021-05-25T00:00:00.000Z"}]
```

**Output**:

```json
[
  {
    "url": "https://www.linkedin.com/posts/altiamkabir_i-asked-chatgpt-to-overhaul-my-linkedin-profile-activity-7380141222167969792-2DQp",
    "id": "7380141222167969792",
    "user_id": "alt***kab***",
    "use_url": "https://bd.linkedin.com/in/altiamkabir?trk=public_post_feed-actor-image",
    "title": "I a***d C***GPT*********aul********************* 7 ***************************************************************************************************************************************************************************",
    "headline": "I a***d C***GPT*********aul*********************",
    "post_text": "I asked ChatGPT to overhaul my LinkedIn profile. 7 simple prompts later — recruiters started finding me: 1/ Profile Audit — Through a Recruiter’s Eyes ↳ “Act as a hiring manager in [industry]. Review my LinkedIn profile summary and tell me exactly what’s weak, unclear, or missing.” 2/ Headline That Gets Found ↳ “Based on my experience in [industry/skills], write a LinkedIn headline that’s keyword-rich, attention-grabbing, and makes me stand out — without sounding like everyone else.” 3/ Summary That Sells (Not Bores) ↳ “Write a compelling LinkedIn summary that showcases my biggest wins, my unique value, and my personality — in 3 short, high-impact paragraphs.” 4/ Experience Section Glow-Up ↳ “Rewrite these 3 job descriptions using action verbs, measurable results, and keywords recruiters in [industry] actually search for.” 5/ Skills Section, Optimized for Hiring ↳ “List the top 10 LinkedIn skills I should add for someone aiming to land a [target role] — based on what hiring managers look for.” 6/ Visibility-Boosting Comment Plan ↳ “Give me a 7-day action plan to comment on posts in my industry that builds credibility, attracts profile visits, and keeps me top-of-mind.” 7/ Thought Leader Mode (Even as a Beginner) ↳ “Write 5 post ideas that share insights, lessons, or tips from my career — so I can look credible without coming across as boastful.” 𝐀𝐈 𝐢𝐬 𝐂𝐡𝐚𝐧𝐠𝐢𝐧𝐠 𝐭𝐡𝐞 𝐖𝐨𝐫𝐥𝐝—𝐀𝐫𝐞 𝐘𝐨𝐮 𝐑𝐞𝐚𝐝𝐲? 🚀 Get $15,000+ in AI knowledge for Zero-cost! ✔️ 60+ ChatGPT chapters ✔️ 38,000+ AI tools ✔️ 600+ AI courses ✔️ 3,000+ prompts Get instant access Here 👉 aiplanetx.com 𝐕𝐢𝐞𝐰 𝐦𝐲 𝐧𝐞𝐰𝐬𝐥𝐞𝐭𝐭𝐞𝐫 from my profile above ⬆️",
    "date_posted": "2025-10-04T07:26:23.728Z",
    "hashtags": null,
    "embedded_links": [
      "http://aiplanetx.com/"
    ],
    "images": [
      "https://media.licdn.com/dms/image/v2/D5622AQFZvZBCFuLFpA/feedshare-shrink_800/B56ZmuHUeBHAAg-/0/1759562779260?e=2147483647&v=beta&t=JY9g5_q2rRYdbnFSa5x_aYIkqSpKxDLgUqPyRdMQLcI"
    ],
    "videos": null,
    "num_likes": 208,
    "num_comments": 42,
    "more_articles_by_user": null,
    "more_relevant_posts": null,
    "top_visible_comments": [
      {
        "comment": "Impressive how AI tools can enhance our professional presence so effectively. Great insights!",
        "comment_date": "2025-10-04T20:21:33.251Z",
        "comment_images": null,
        "num_reactions": 2,
        "tagged_users": null,
        "use_url": "https://www.linkedin.com/company/ai-chatgpt-prompts?trk=public_post_comment_actor-name",
        "user_id": "ai-chatgpt-prompts",
        "user_name": "AI: Artificial Intelligence",
        "user_title": null
      },
      {
        "comment": "Wow. This is very helpful Altiam Kabir",
        "comment_date": "2025-10-04T20:24:33.253Z",
        "comment_images": null,
        "num_reactions": 2,
        "tagged_users": [
          "https://bd.linkedin.com/in/altiamkabir?trk=public_post_comment-text"
        ],
        "use_url": "https://sg.linkedin.com/in/alexey6?trk=public_post_comment_actor-name",
        "user_id": "alexey6",
        "user_name": "Alexey Navolokin",
        "user_title": "FOL*** ME***r b*********ech*********************pin***************************************************************************"
      },
      {
        "comment": "Incredible insights! AI tools are proving invaluable in enhancing our professional visibility. Grateful for the guidance provided.",
        "comment_date": "2025-10-04T20:21:33.256Z",
        "comment_images": null,
        "num_reactions": 2,
        "tagged_users": null,
        "use_url": "https://www.linkedin.com/company/ai-prompts-community?trk=public_post_comment_actor-name",
        "user_id": "ai-prompts-community",
        "user_name": "AI & Prompts Hub",
        "user_title": null
      },
      {
        "comment": "This shows how leveraging AI strategically can open doors quickly. Altiam Kabir",
        "comment_date": "2025-10-04T20:24:33.258Z",
        "comment_images": null,
        "num_reactions": 3,
        "tagged_users": [
          "https://bd.linkedin.com/in/altiamkabir?trk=public_post_comment-text"
        ],
        "use_url": "https://ie.linkedin.com/in/awa-k-penn?trk=public_post_comment_actor-name",
        "user_id": "awa-k-penn",
        "user_name": "Awa K. Penn",
        "user_title": "AI ***t P***els********* Be***************"
      },
      {
        "comment": "This is such a practical way to use AI for career growth Altiam Kabir",
        "comment_date": "2025-10-04T20:24:33.322Z",
        "comment_images": null,
        "num_reactions": 5,
        "tagged_users": [
          "https://bd.linkedin.com/in/altiamkabir?trk=public_post_comment-text"
        ],
        "use_url": "https://www.linkedin.com/in/gavriel-blaxberg?trk=public_post_comment_actor-name",
        "user_id": "gavriel-blaxberg",
        "user_name": "Gav Blaxberg",
        "user_title": "CEO*** WO***Fin*********#1 *********************or ***************************************************************************************************************************************************"
      },
      {
        "comment": "Amazing share. Happy weekend Altiam Kabir",
        "comment_date": "2025-10-04T20:23:33.324Z",
        "comment_images": null,
        "num_reactions": 4,
        "tagged_users": [
          "https://bd.linkedin.com/in/altiamkabir?trk=public_post_comment-text"
        ],
        "use_url": "https://ng.linkedin.com/in/cordelia-jiakponna-b-sc-mba-acis-5ab58b29?trk=public_post_comment_actor-name",
        "user_id": "cordelia-jiakponna-b-sc-mba-acis-5ab58b29",
        "user_name": "Cordelia Jiakponna, B.Sc., MBA, ACIS",
        "user_title": "Hel***g E***uti*********ply********************* In*********************************************************************************************"
      },
      {
        "comment": "Great playbook, very practical and clear. How did you track changes in recruiter responses after using these prompts and can you share any metrics or examples, and a quick tip is to track profile views and inbound messages week over week.",
        "comment_date": "2025-10-04T20:23:33.326Z",
        "comment_images": null,
        "num_reactions": 4,
        "tagged_users": null,
        "use_url": "https://www.linkedin.com/in/drew-thomas-oneiro-tech?trk=public_post_comment_actor-name",
        "user_id": "drew-thomas-oneiro-tech",
        "user_name": "Drew Thomas",
        "user_title": "CEO***One*** Te*********s |*********************rk ***************************************************************************************************************************************"
      },
      {
        "comment": "A powerful example of how strategic AI prompting can turn LinkedIn from a static profile into an active opportunity generator.",
        "comment_date": "2025-10-04T20:24:33.328Z",
        "comment_images": null,
        "num_reactions": 2,
        "tagged_users": null,
        "use_url": "https://pk.linkedin.com/in/ahmed-zia-siddiqi-894958200?trk=public_post_comment_actor-name",
        "user_id": "ahmed-zia-siddiqi-894958200",
        "user_name": "Ahmed Zia Siddiqi",
        "user_title": "I S*** Yo***Exe*********ile*********************est******************************************************************************************************************************"
      },
      {
        "comment": "These AI-driven strategies to optimize LinkedIn profiles are a game-changer for job seekers. Thanks for sharing!",
        "comment_date": "2025-10-04T20:21:33.329Z",
        "comment_images": null,
        "num_reactions": 1,
        "tagged_users": null,
        "use_url": "https://www.linkedin.com/company/ai-usecase?trk=public_post_comment_actor-name",
        "user_id": "ai-usecase",
        "user_name": "AI Use Cases",
        "user_title": null
      },
      {
        "comment": "Your experience showcases the potential of AI tools in modern career development, which is exciting! I’ve seen others have success with similar strategies.",
        "comment_date": "2025-10-04T20:25:33.330Z",
        "comment_images": null,
        "num_reactions": 3,
        "tagged_users": null,
        "use_url": "https://www.linkedin.com/in/suepats?trk=public_post_comment_actor-name",
        "user_id": "suepats",
        "user_name": "Sumedha Patwardhan (Sue Pats)",
        "user_title": "R.A***I.D***eve*********rin*********************ys-*********************************************************************************************************************************************************************"
      }
    ],
    "user_followers": 105139,
    "user_posts": 2002,
    "user_articles": 6,
    "post_type": "post",
    "account_type": "Person",
    "post_text_html": "I asked ChatGPT to overhaul my LinkedIn profile.<br/><br/>7 simple prompts later &#x2014; recruiters started finding me:<br/><br/><br/>1/ Profile Audit &#x2014; Through a Recruiter&#x2019;s Eyes<br/><br/>&#x21B3; &#x201C;Act as a hiring manager in [industry]. Review my LinkedIn profile summary and tell me exactly what&#x2019;s weak, unclear, or missing.&#x201D;<br/><br/><br/>2/ Headline That Gets Found<br/><br/>&#x21B3; &#x201C;Based on my experience in [industry/skills], write a LinkedIn headline that&#x2019;s keyword-rich, attention-grabbing, and makes me stand out &#x2014; without sounding like everyone else.&#x201D;<br/><br/><br/>3/ Summary That Sells (Not Bores)<br/><br/>&#x21B3; &#x201C;Write a compelling LinkedIn summary that showcases my biggest wins, my unique value, and my personality &#x2014; in 3 short, high-impact paragraphs.&#x201D;<br/><br/><br/>4/ Experience Section Glow-Up<br/><br/>&#x21B3; &#x201C;Rewrite these 3 job descriptions using action verbs, measurable results, and keywords recruiters in [industry] actually search for.&#x201D;<br/><br/><br/>5/ Skills Section, Optimized for Hiring<br/><br/>&#x21B3; &#x201C;List the top 10 LinkedIn skills I should add for someone aiming to land a [target role] &#x2014; based on what hiring managers look for.&#x201D;<br/><br/><br/>6/ Visibility-Boosting Comment Plan<br/><br/>&#x21B3; &#x201C;Give me a 7-day action plan to comment on posts in my industry that builds credibility, attracts profile visits, and keeps me top-of-mind.&#x201D;<br/><br/><br/>7/ Thought Leader Mode (Even as a Beginner)<br/><br/>&#x21B3; &#x201C;Write 5 post ideas that share insights, lessons, or tips from my career &#x2014; so I can look credible without coming across as boastful.&#x201D;<br/><br/><br/><br/>&#x1D400;&#x1D408; &#x1D422;&#x1D42C; &#x1D402;&#x1D421;&#x1D41A;&#x1D427;&#x1D420;&#x1D422;&#x1D427;&#x1D420; &#x1D42D;&#x1D421;&#x1D41E; &#x1D416;&#x1D428;&#x1D42B;&#x1D425;&#x1D41D;&#x2014;&#x1D400;&#x1D42B;&#x1D41E; &#x1D418;&#x1D428;&#x1D42E; &#x1D411;&#x1D41E;&#x1D41A;&#x1D41D;&#x1D432;?<br/>&#x1F680; Get $15,000+ in AI knowledge for Zero-cost!<br/><br/>&#x2714;&#xFE0F; 60+ ChatGPT chapters<br/>&#x2714;&#xFE0F; 38,000+ AI tools<br/>&#x2714;&#xFE0F; 600+ AI courses<br/>&#x2714;&#xFE0F; 3,000+ prompts<br/><br/>Get instant access Here &#x1F449; <a class=\"link\" href=\"https://www.linkedin.com/redir/redirect?url=http%3A%2F%2Faiplanetx%2Ecom&amp;urlhash=5sqA&amp;trk=public_post-text\" target=\"_self\" rel=\"nofollow\" data-tracking-control-name=\"public_post-text\" data-tracking-will-navigate>aiplanetx.com</a>&#xA0;<br/>&#x1D415;&#x1D422;&#x1D41E;&#x1D430; &#x1D426;&#x1D432; &#x1D427;&#x1D41E;&#x1D430;&#x1D42C;&#x1D425;&#x1D41E;&#x1D42D;&#x1D42D;&#x1D41E;&#x1D42B; from my profile above &#x2B06;&#xFE0F;",
    "repost": {
      "repost_attachments": null,
      "repost_date": null,
      "repost_hangtags": null,
      "repost_id": null,
      "repost_text": null,
      "repost_url": null,
      "repost_user_id": null,
      "repost_user_name": null,
      "repost_user_title": null,
      "tagged_companies": null,
      "tagged_users": null
    },
    "tagged_companies": [],
    "tagged_people": [],
    "user_title": "AI ***cat***| B*********0K+*********************abo************************************************************************************",
    "author_profile_pic": "htt***//m***a.l*********dms*********************E5e*********************************************************************************************************************************************",
    "num_connections": null,
    "video_duration": null,
    "external_link_data": null,
    "video_thumbnail": null,
    "document_cover_image": null,
    "document_page_count": null
  }
]
```


---
## 8) Email extraction 

### 8.1) APIfy bhansalisoft/linkedin-email-scraper

**Назначение.** Запуск актора `bhansalisoft~linkedin-email-scraper`, который собирает e-mail’ы из профилей LinkedIn через Google-поиск. Три режима вызова:

**Endpoint**: `1KoCTvsASIAXqkvY7`


**Вход (application/json, схема `inputSchema`):**

```json
{
  "Keyword": "site:linkedin.com/in \"VP Engineering\" AND Stripe",
  "location": "San Francisco",
  "social_network": "linkedin.com/",
  "Country": "www",
  "Email_Type": "0",
  "Other_Email_Type": "@stripe.com",
  "proxySettings": {
    "useApifyProxy": true
  }
}
```

Поля:

* **Keyword** *(string, required)* — поисковый запрос (обычно Google dork под LinkedIn профили).
* **location** *(string, optional)* — уточняющая локация.
* **social_network** *(enum, required, default `linkedin.com/`)* — целевая сеть.
* **Country** *(enum, required, default `www`)* — регион поиска (двухбуквенный код списка актора).
* **Email_Type** *(enum: `"0"|"1"`, required)* — режим поиска e-mail (зависит от реализации актора).
* **Other_Email_Type** *(string, optional)* — если нужен фильтр по домену, напр. `"@domain.com"`.
* **proxySettings** *(object, optional)* — конфиг прокси Apify (или свой), если требуется.

**Выход (200):** Массив dataset items, структура зависит от актора. Типично:

```json
[
  {
    "fullName": "Jane Doe",
    "profileUrl": "https://www.linkedin.com/in/janedoe/",
    "title": "VP Engineering at Stripe",
    "email": "jane@stripe.com",
    "source": "linkedin|website|guess",
    "company": "Stripe",
    "location": "San Francisco, CA"
  }
]
```

### 8.2) APIfy dev_fusion/Linkedin-Profile-Scraper

**Endpoint**: `2SyF0bVxmgGr8IVCZ`

**Input**
```json
{
    "profileUrls": [
        "https://www.linkedin.com/in/williamhgates",
        "http://www.linkedin.com/in/jeannie-wyrick-b4760710a",
        "https://www.linkedin.com/in/ricky-hoover-7388371b8"
    ]
}
```

**Output**
```json
[{
    "linkedinUrl": "https://www.linkedin.com/in/ricky-hoover-7388371b8",
    "firstName": "Ricky",
    "lastName": "Hoover",
    "fullName": "Ricky Hoover",
    "headline": "Sales Development @project44",
    "connections": 2451,
    "followers": 2726,
    "email": "ricky.hoover@project44.com",
    "mobileNumber": null,
    "jobTitle": "Sales Development Representative",
    "companyName": "Project44",
    "companyIndustry": "Logistics And Supply Chain",
    "companyWebsite": "project44.com",
    "companyLinkedin": "linkedin.com/company/project-44",
    "companyFoundedIn": 2014,
    "companySize": "501-1000",
    "currentJobDuration": "1 yr 6 mos",
    "currentJobDurationInYrs": 1.5,
    "topSkillsByEndorsements": "Nonprofit Organizations",
    "addressCountryOnly": "United States",
    "addressWithCountry": "Chicago, Illinois, United States",
    "addressWithoutCountry": "Chicago, Illinois",
    "profilePic": "https://media.licdn.com/dms/image/v2/D5603AQHejeBBww5rRw/profile-displayphoto-shrink_100_100/profile-displayphoto-shrink_100_100/0/1726602865808?e=1762387200&v=beta&t=OKjsnwnCYZrHN9TUARdmemR32-57jDGAsTzXv--HhD4",
    "profilePicHighQuality": "https://media.licdn.com/dms/image/v2/D5603AQHejeBBww5rRw/profile-displayphoto-shrink_800_800/profile-displayphoto-shrink_800_800/0/1726602865831?e=1762387200&v=beta&t=fTW9ZlNNxQyGqrdp0EbBy2dXV10KjpVdnRvBvLQCTmw",
    "about": null,
    "publicIdentifier": "ricky-hoover-7388371b8",
    "openConnection": null,
    "urn": "ACoAADKrRlwBHYP0WK6RYyFU2rQqC7b4dPfCAUQ",
    "experiences": [
      {
        "companyId": "9265792",
        "companyUrn": "urn:li:fsd_company:9265792",
        "companyLink1": "https://www.linkedin.com/company/9265792/",
        "logo": "https://media.licdn.com/dms/image/v2/D560BAQG0esieZ4s8pg/company-logo_200_200/company-logo_200_200/0/1719842515839/project_44_logo?e=1762387200&v=beta&t=Kyjwkf4Guzv6WPvqf7Sfd38lv44lYUrixLn5M0yxBkU",
        "title": "Sales Development Representative",
        "subtitle": "project44 · Full-time",
        "caption": "May 2024 - Present · 1 yr 6 mos",
        "metadata": "Chicago, Illinois, United States · Hybrid",
        "breakdown": false,
        "subComponents": [
          {
            "description": []
          }
        ]
      },
      {
        "companyId": "24440",
        "companyUrn": "urn:li:fsd_company:24440",
        "companyLink1": "https://www.linkedin.com/company/24440/",
        "logo": "https://media.licdn.com/dms/image/v2/D4E0BAQFxLQ6HpbS3WA/company-logo_200_200/company-logo_200_200/0/1702053328104/collabera_logo?e=1762387200&v=beta&t=va1sHaNE5OXfVTEGTVsReOGpef2DpZbsyG61Otj5D5Y",
        "title": "Collabera",
        "subtitle": "Full-time · 1 yr",
        "caption": "Milwaukee, Wisconsin, United States",
        "breakdown": true,
        "subComponents": [
          {
            "title": "Account Manager",
            "caption": "Feb 2024 - May 2024 · 4 mos",
            "metadata": "On-site",
            "description": []
          },
          {
            "title": "Associate",
            "caption": "Jun 2023 - Jan 2024 · 8 mos",
            "description": []
          }
        ]
      },
      {
        "companyId": "18958",
        "companyUrn": "urn:li:fsd_company:18958",
        "companyLink1": "https://www.linkedin.com/company/18958/",
        "logo": "https://media.licdn.com/dms/image/v2/D4E0BAQEuhFoTJO4uEw/company-logo_100_100/B4EZfFA06SGcAQ-/0/1751357018130/bostik_logo?e=1762387200&v=beta&t=Kb31NYckx1s50UT6MCIhPzx-iK8h7-z0vRyt4ESE0V0",
        "title": "Sales And Marketing Intern",
        "subtitle": "Bostik · Internship",
        "caption": "Apr 2022 - May 2023 · 1 yr 2 mos",
        "metadata": "Milwaukee, Wisconsin, United States",
        "breakdown": false,
        "subComponents": [
          {
            "description": []
          }
        ]
      },
      {
        "companyId": "968501",
        "companyUrn": "urn:li:fsd_company:968501",
        "companyLink1": "https://www.linkedin.com/company/968501/",
        "logo": "https://media.licdn.com/dms/image/v2/D4E0BAQHt81qAqgkxQA/company-logo_100_100/B4EZcIqk_aHkAQ-/0/1748197069133/mad_science_logo?e=1762387200&v=beta&t=ceF8ZeCajhUWQWC7NvYXfqdJ457G9NSwOaUwqKSWJB0",
        "title": "Course Instructor",
        "subtitle": "Mad Science · Part-time",
        "caption": "Mar 2021 - Feb 2022 · 1 yr",
        "metadata": "Milwaukee, Wisconsin, United States",
        "breakdown": false,
        "subComponents": [
          {
            "description": [
              {
                "type": "textComponent",
                "text": "Running Science camps for elementary to middle school level children"
              }
            ]
          }
        ]
      },
      {
        "companyId": "3605277",
        "companyUrn": "urn:li:fsd_company:3605277",
        "companyLink1": "https://www.linkedin.com/company/3605277/",
        "logo": "https://media.licdn.com/dms/image/v2/C4D0BAQFznzS8rCnkQw/company-logo_200_200/company-logo_200_200/0/1631331338419?e=1762387200&v=beta&t=LDXf9Tzo1aanUYZur_M10sll7N--QxG2Im8TyC3n66o",
        "title": "Tutor",
        "subtitle": "Summit Educational Association · Part-time",
        "caption": "Aug 2019 - Feb 2021 · 1 yr 7 mos",
        "metadata": "Milwaukee, Wisconsin, United States",
        "breakdown": false,
        "subComponents": [
          {
            "description": [
              {
                "type": "textComponent",
                "text": "Mentoring and tutoring elementary school students Math and English skills"
              }
            ]
          }
        ]
      }
    ],
    "updates": [],
    "skills": [
      {
        "title": "Educational Technology",
        "subComponents": [
          {
            "description": []
          }
        ]
      },
      {
        "title": "Nonprofit Organizations",
        "subComponents": [
          {
            "description": [
              {
                "type": "insightComponent",
                "text": "Endorsed by 1 person in the last 6 months"
              },
              {
                "type": "insightComponent",
                "text": "3 endorsements"
              }
            ]
          }
        ]
      },
      {
        "title": "Search Engine Optimization (SEO)",
        "subComponents": [
          {
            "description": []
          }
        ]
      },
      {
        "title": "Promo",
        "subComponents": [
          {
            "description": []
          }
        ]
      },
      {
        "title": "Sales Prospecting",
        "subComponents": [
          {
            "description": []
          }
        ]
      },
      {
        "title": "Healthcare Delivery",
        "subComponents": [
          {
            "description": []
          }
        ]
      },
      {
        "title": "Media Relations",
        "subComponents": [
          {
            "description": []
          }
        ]
      },
      {
        "title": "Media Buying",
        "subComponents": [
          {
            "description": []
          }
        ]
      },
      {
        "title": "Relationship Building",
        "subComponents": [
          {
            "description": []
          }
        ]
      },
      {
        "title": "Social Influence",
        "subComponents": [
          {
            "description": []
          }
        ]
      },
      {
        "title": "Computer Literacy",
        "subComponents": [
          {
            "description": []
          }
        ]
      },
      {
        "title": "Merchandising",
        "subComponents": [
          {
            "description": []
          }
        ]
      },
      {
        "title": "Client Relations",
        "subComponents": [
          {
            "description": []
          }
        ]
      },
      {
        "title": "Customer Relationship Management (CRM)",
        "subComponents": [
          {
            "description": []
          }
        ]
      },
      {
        "title": "Proposal Writing",
        "subComponents": [
          {
            "description": []
          }
        ]
      },
      {
        "title": "Technical Recruiting",
        "subComponents": [
          {
            "description": []
          }
        ]
      },
      {
        "title": "Cold Calling",
        "subComponents": [
          {
            "description": []
          }
        ]
      },
      {
        "title": "Account Coordination",
        "subComponents": [
          {
            "description": []
          }
        ]
      },
      {
        "title": "Direct Sales",
        "subComponents": [
          {
            "description": []
          }
        ]
      },
      {
        "title": "Business-to-Business (B2B)",
        "subComponents": [
          {
            "description": []
          }
        ]
      }
    ],
    "profilePicAllDimensions": [
      {
        "width": 100,
        "expiresAt": 1762387200000,
        "height": 100,
        "url": "https://media.licdn.com/dms/image/v2/D5603AQHejeBBww5rRw/profile-displayphoto-shrink_100_100/profile-displayphoto-shrink_100_100/0/1726602865808?e=1762387200&v=beta&t=OKjsnwnCYZrHN9TUARdmemR32-57jDGAsTzXv--HhD4"
      },
      {
        "width": 200,
        "expiresAt": 1762387200000,
        "height": 200,
        "url": "https://media.licdn.com/dms/image/v2/D5603AQHejeBBww5rRw/profile-displayphoto-shrink_200_200/profile-displayphoto-shrink_200_200/0/1726602865808?e=1762387200&v=beta&t=M6ILUpJCECgUyzfoWvvdcffASU_7sAaKbiU6yVqmIFk"
      },
      {
        "width": 400,
        "expiresAt": 1762387200000,
        "height": 400,
        "url": "https://media.licdn.com/dms/image/v2/D5603AQHejeBBww5rRw/profile-displayphoto-shrink_400_400/profile-displayphoto-shrink_400_400/0/1726602865808?e=1762387200&v=beta&t=DYHuTBIhJxv-sGe6dcOrGyPm9HUrYl8Ix_yqf-PZzUc"
      },
      {
        "width": 800,
        "expiresAt": 1762387200000,
        "height": 800,
        "url": "https://media.licdn.com/dms/image/v2/D5603AQHejeBBww5rRw/profile-displayphoto-shrink_800_800/profile-displayphoto-shrink_800_800/0/1726602865831?e=1762387200&v=beta&t=fTW9ZlNNxQyGqrdp0EbBy2dXV10KjpVdnRvBvLQCTmw"
      }
    ],
    "educations": [
      {
        "companyId": "7160",
        "companyUrn": "urn:li:fsd_company:7160",
        "companyLink1": "https://www.linkedin.com/company/7160/",
        "logo": "https://media.licdn.com/dms/image/v2/D560BAQEeeWyCdj-Eeg/company-logo_200_200/company-logo_200_200/0/1718972537090/marquetteu_logo?e=1762387200&v=beta&t=Gqxhrdm39T0bkMau9Ahg0hQFuVCm0wbXzpWJgX2Q_As",
        "title": "Marquette University",
        "subtitle": "Business Communication",
        "breakdown": false,
        "subComponents": [
          {
            "description": []
          }
        ]
      }
    ],
    "licenseAndCertificates": [],
    "honorsAndAwards": [],
    "languages": [],
    "volunteerAndAwards": [],
    "verifications": [],
    "promos": [],
    "highlights": [],
    "projects": [],
    "publications": [],
    "patents": [],
    "courses": [],
    "testScores": [],
    "organizations": [],
    "volunteerCauses": [],
    "interests": [],
    "recommendations": []
  }]
```

### 8.3) Hunter.io

**Назначение.** Добыть контакты/почты из профилей/сайтов; где возможно — валидация.

**Офиц. доки:**
• Hunter API (Domain Search, Finder, Verifier). ([Hunter][6])

**Наш эндпойнт.**
`GET https://api.hunter.io/v2/domain-search?domain=intercom.com`
`GET https://api.hunter.io/v2/email-finder?domain=reddit.com&first_name=Alexis`

**Output 1**

```json
{
  "data": {
    "domain": "intercom.com",
    "disposable": false,
    "webmail": false,
    "accept_all": true,
    "pattern": "{first}",
    "organization": "Intercom",
    "emails": [
      {
        "value": "ciaran@intercom.com",
        "type": "personal",
        "confidence": 92,
        "sources": [
          {
            "domain": "github.com",
            "uri": "http://github.com/ciaranlee",
            "extracted_on": "2015-07-29",
            "last_seen_on": "2017-07-01",
            "still_on_page": true
          },
          {
            "domain": "blog.intercom.com",
            "uri": "http://blog.intercom.com/were-hiring-a-support-engineer/",
            "extracted_on": "2015-08-29",
            "last_seen_on": "2017-07-01",
            "still_on_page": true
          },
          ...
        ],
        "first_name": "Ciaran",
        "last_name": "Lee",
        "position": "Support Engineer",
        "position_raw": "Support Engineer",
        "seniority": "senior",
        "department": "it",
        "linkedin": null,
        "twitter": "ciaran_lee",
        "phone_number": null,
        "verification": {
          "date": "2019-12-06",
          "status": "valid"
        }
      },
      ...
    ],
    "linked_domains": []
  },
  "meta": {
    "results": 35,
    "limit": 10,
    "offset": 0,
    "params": {
      "domain": "intercom.com",
      "company": null,
      "type": null,
      "seniority": null,
      "department": null
    }
  }
}
```

**Output 2**

```json
{
  "data": {
    "first_name": "Alexis",
    "last_name": "Ohanian",
    "email": "alexis@reddit.com",
    "score": 97,
    "domain": "reddit.com",
    "accept_all": false,
    "position": "Cofounder",
    "twitter": null,
    "linkedin_url": null,
    "phone_number": null,
    "company": "Reddit",
    "sources": [
      {
        "domain": "redditblog.com",
        "uri": "http://redditblog.com/2008/10/22/widgets-get-an-upgrade-and-a-firefox-extension-that-will-rock-your-world",
        "extracted_on": "2018-10-19",
        "last_seen_on": "2021-05-18",
        "still_on_page": true
      },
      ...
    ],
    "verification": {
      "date": "2021-06-14",
      "status": "valid"
    }
  },
  "meta": {
    "params": {
      "first_name": "Alexis",
      "last_name": "Ohanian",
      "full_name": null,
      "domain": "reddit.com",
      "company": null,
      "max_duration": null
    }
  }
}
```
---

## 9) Валидация emails

**Описание:** Используем 2-3 уровневый пайплайн. Сначала валидируем через локально развернутый ReacherHQ и затем идем в neverbounce и проверяем там email, в конце мы можем сходить в hunter.io (опционально) и затем посчитать итоговую вероятность, что этот email реально валидный. 

Бери адаптивные веса: NeverBounce обычно точнее live-SMTP локальных валидаторов на массовых доменах, но Reacher даёт богатые сабсигналы. Hunter — хороший дооценщик.
Базовые веса:
- w_reacher = 0.35
- w_nb = 0.50
- w_hunter = 0.15 (если есть)

Адаптация весов:
- Если NeverBounce.result == valid → уменьшай w_nb до 0.35, остальное пропорционально вверх.
- Если у Reacher smtp.is_reachable==true → w_reacher += 0.10 (и нормализуй).
- Если Hunter отсутствует → перекинь его вес пропорционально на другие.

Итоговый скор считаем по формуле: `p_base = (w_reacher * p_reacher + w_nb * p_nb + w_hunter * p_hunter) / (w_reacher + w_nb + w_hunter)`

### 9.1) Reach

**Endpoint**: `POST http://localhost:8080/v0/check_email`

**Input** 
```json
{
    "to_email": "someone@gmail.com",
    "proxy": {                        // (optional) SOCK5 proxy to run the verification through, default is empty
        "host": "my-proxy.io",
        "port": 1080,
        "username": "me",             // (optional) Proxy username
        "password": "pass"            // (optional) Proxy password
    },
}
```
**Output**
```json
{
	"input": "someone@gmail.com",
	"is_reachable": "invalid",
	"misc": {
		"is_disposable": false,
		"is_role_account": false,
		"is_b2c": true
	},
	"mx": {
		"accepts_mail": true,
		"records": [
			"alt3.gmail-smtp-in.l.google.com.",
			"gmail-smtp-in.l.google.com.",
			"alt1.gmail-smtp-in.l.google.com.",
			"alt4.gmail-smtp-in.l.google.com.",
			"alt2.gmail-smtp-in.l.google.com."
		]
	},
	"smtp": {
		"can_connect_smtp": true,
		"has_full_inbox": false,
		"is_catch_all": false,
		"is_deliverable": false,
		"is_disabled": true
	},
	"syntax": {
		"domain": "gmail.com",
		"is_valid_syntax": true,
		"username": "someone",
		"suggestion": null
	}
}
```

### 9.2) NeverBounce

**Endpoint**: `POST https://api.neverbounce.com/v4/single/check?key={api_key}&email={email}`

**Output**
```json
{
  "status": "success",
  "result": "valid",
  "flags": [
    "has_dns",
    "has_dns_mx"
  ],
  "suggested_correction": "",
  "retry_token": "",
  "execution_time": 499
}
```
### 9.3) Hunter.io email validation

**Endpoint**: `GET https://api.hunter.io/v2/email-verifier?email=patrick@stripe.com`

**Output**

```json
{
  "data": {
    "status": "valid",
    "result": "deliverable",
    "_deprecation_notice": "Using result is deprecated, use status instead",
    "score": 100,
    "email": "patrick@stripe.com",
    "regexp": true,
    "gibberish": false,
    "disposable": false,
    "webmail": false,
    "mx_records": true,
    "smtp_server": true,
    "smtp_check": true,
    "accept_all": false,
    "block": false,
    "sources": [
      {
        "domain": "beta.paganresearch.io",
        "uri": "http://beta.paganresearch.io/details/stripe",
        "extracted_on": "2020-06-17",
        "last_seen_on": "2020-06-17",
        "still_on_page": true
      },
      {
        "domain": "icloudnewz.blogspot.com",
        "uri": "http://icloudnewz.blogspot.com/2017/11/follow-patrick-collison-mike-birbiglia.html",
        "extracted_on": "2020-03-25",
        "last_seen_on": "2020-06-29",
        "still_on_page": true
      }
    ]
  },
  "meta": {
    "params": {
      "email": "patrick@stripe.com"
    }
  }
}
```
---
## 10) Qualify service

**Description**: Сравнивает компанию с нашим ICP. Self hosted


**Input**:

```json


{
  "idempotency_key": "hash(project44.com|2025-10-02)",
  "fetched_at": "2025-10-02T06:20:00Z",
  "primary_domain": "project44.com",

  "company": {
    "name": "project44",
    "country_code": "US",
    "website_url": "https://www.project44.com",
    "linkedin_url": "https://www.linkedin.com/company/project-44/",
    "language": "en"
  },

  "signals": {
    "website": {
      "title": "Movement. The Decision Intelligence Platform...",
      "content_markdown": "<очищенный markdown>",
      "headings": [{"level": 1, "text": "Decision Intelligence Platform"}],
      "metrics": { "char_len": 24500, "link_ratio": 0.08, "estimated_read_time_sec": 540 }
    },
    "linkedin": {
      "id": "123456",
      "followers": 120000,
      "employees_in_linkedin": 1800,
      "about": "project44 provides real-time visibility...",
      "industries": "Software Development; Transportation/Logistics",
      "company_size": "1001-5000",
      "headquarters": "Chicago, Illinois"
    },
    "google": {
      "keyword": "project44 supply chain",
      "page_title": "Decision Intelligence Platform | project44",
      "organic_top": [
        { "rank": 1, "title": "project44 — Movement", "url": "https://www.project44.com", "description": "Real-time visibility..." },
        { "rank": 2, "title": "About project44", "url": "https://www.project44.com/company", "description": "About p44..." }
      ],
      "related": ["supply chain visibility", "real time tracking"]
    }
  },

  "embedding": {
    "text": "title + H1..H3 + LI about + 2-3 organic descriptions",
    "model": "text-embedding-3-large",
    "vector": null
  },

  "icp": {
    "icp_id": "global.logistics_smb",
    "title": "Logistics — SMB",
    "market": "global",

    "include": {
      "industries": ["Logistics", "Supply Chain", "Transportation"],
      "countries": ["US", "GB", "DE"],
      "size_band": [{"50":"1000"}],
      "keywords": ["visibility", "tracking", "supply chain"],
      "ban_words": ["porn", "tabacco"],
      "description": "We need company with size 1,2,3 etc."
    },

    "scoring": {
      "embedding_model": "bge-m3",
      "alpha": 0.7,
      "beta": 0.3,
      "candidate_labels": ["supply_chain_platform", "ai_analytics_for_logistics", "not_a_fit"]
    }
  },

  "rules":{
   "include_words": ["word1"],
   "exclude_words": ["word2"],
   "employees_in_linkedin": {"min": 1800, "max": 5000}, //optional
  },

  "raw": {
    "google": { "…": "усечённый сырой json" },
    "linkedin": { "…": "усечённый сырой объект" },
    "website": {
      "cleaner_version": "llm:v1",
      "raw_content_sample": "часть raw для аудита"
    }
  }
}
```

**Output**:

```json
{
  "company_id": "hash(project44.com|project44)",
  "name": "project44",
  "primary_domain": "project44.com",
  "industry": "Logistics / Supply Chain",
  "classification": {
    "label": "supply_chain_platform",
    "confidence": 0.91,
    "method": "rule+embedding+llm",
    "explanation": "Detected keywords: 'visibility', 'supply chain AI'"
  },
  "fingerprint": {
    "top_terms": [
      {"term": "visibility", "score": 0.85},
      {"term": "supply chain", "score": 0.82},
      {"term": "logistics", "score": 0.77},
      {"term": "ai disruption", "score": 0.62},
      {"term": "decision intelligence", "score": 0.59}
    ],
    "distinctive_terms": [
      {"term": "tariff analytics", "delta": 0.45},
      {"term": "yard automation", "delta": 0.41},
      {"term": "port intel", "delta": 0.36}
    ],
    "assigned_tags": [
      "Logistics",
      "Visibility",
      "AI/Analytics"
    ]
  },
  "signals": {
    "followers": 120000,
    "employees_in_linkedin": 1800,
    "content_score": 0.82,
    "language": "en"
  },
  "embedding": {
    "model": "text-embedding-3-large",
    "vector": null
  },
  "company": "same as input data",
  "timestamp": "2025-10-02T07:30:00Z",
  "version": "classifier:v2"
}
```
