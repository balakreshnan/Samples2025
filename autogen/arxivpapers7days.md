# Summarize arxiv papers from the last 7 days

## Introduction

- A system to summarize arxiv papers from the last 7 days.
- Using Autogen Studio as UI to summarize the papers.
- Using tools to dynamicsally fetch the latest papers.
- Getting last 7 days papers from arxiv.
- Will current day and then go back 7 days.
- Only for knowledge purposes.
- Not for production use.
- Using Autogen 0.5 and above.
- Using Azure Open AI service to run the model.
- Using gpt 4.1 model.

## Steps

- First install the Autogen Studio.
- also install autogen package.

```
autogen-agentchat
autogenstudio
autogen-ext
autogen-ext[openai]
autogen-ext[azure]
openai
uv
```

- Create a new Team in Autogen Studio using UI
- we can run the studio using the following command

```
autogenstudio ui --port 8081
```

- Create a new team and add the following systems to the team
- Here is the overall flow of the team

![info](https://github.com/balakreshnan/Samples2025/blob/main/autogen/images/arxivpap-3.jpg 'RagChat')

- System prompt for Arxiv Paper Fetcher:

```
You are a helpful assistant. Solve tasks carefully. When done, say TERMINATE.
```

- Create the tool to fetch the latest arxiv papers from the last 7 days

```python
def arxivfunction():
    base_url = 'http://export.arxiv.org/api/query?'
    categories=['cs', 'stat', 'math']
    max_results = 100
    days_back=7

    # Build category query (e.g., "cat:cs* OR cat:stat*")
    category_query = ' OR '.join([f'cat:{cat}*' for cat in categories])
    encoded_query = quote(category_query)

    # Build the full URL
    query_url = (f"{base_url}search_query={encoded_query}"
                 f"&start=0&max_results={max_results}&sortBy=submittedDate&sortOrder=descending")

    print(f"Fetching from: {query_url}")

    # Parse feed
    feed = feedparser.parse(query_url)

    # Filter results by date
    cutoff_date = datetime.utcnow() - timedelta(days=days_back)
    recent_papers = []

    for entry in feed.entries:
        published_date = datetime.strptime(entry.published, '%Y-%m-%dT%H:%M:%SZ')
        if published_date >= cutoff_date:
            pdf_url = next((link.href for link in entry.links if link.type == 'application/pdf'), None)
            recent_papers.append({
                'title': entry.title.strip(),
                'authors': [author.name for author in entry.authors],
                'summary': entry.summary.strip(),
                'published': published_date.strftime('%Y-%m-%d'),
                'pdf_url': pdf_url,
                'arxiv_url': entry.link
            })

    return recent_papers
```

- Add the tool to the team
- Make sure you add the imports

![info](https://github.com/balakreshnan/Samples2025/blob/main/autogen/images/arxivpap-4.jpg 'RagChat')

- now add another assistant to Send Emails
- Here is the system prompt for Send Email:

```
You are a Email AI assistant. Get the content from arxivagent and send email to given email address using tools. Solve tasks carefully. When done, say TERMINATE.
```

- Make sure you add the imports

![info](https://github.com/balakreshnan/Samples2025/blob/main/autogen/images/arxivpap-5.jpg 'RagChat')

- Here is the code to send email

```python
# Create an email-sending function
def send_email(to_emails: str, subject: str, body: str) -> str:
    # Email configuration (replace with your SMTP server details)
    print("Sending email...")
    smtp_server = "smtp.gmail.com"
    smtp_port = 587
    sender_email = os.getenv("GOOGLE_EMAIL")
    sender_password = os.getenv("GOOGLE_APP_PASSWORD")

    # Split the CSV string into a list of email addresses
    recipients = [email.strip() for email in to_emails.split(",")]

    # Create the email message
    message = MIMEMultipart()
    message["From"] = sender_email
    # message["To"] = to_email
    message["Subject"] = subject
    message.attach(MIMEText(body, "plain"))


    # Send the email
    with smtplib.SMTP(smtp_server, smtp_port) as server:
        server.starttls()
        server.login(sender_email, sender_password)
        for recipient in recipients:
            message["To"] = recipient
            server.send_message(message)
            print(f"Email sent to {recipient} with subject: {subject}")
    
    print(f"Email sent to {to_emails} with subject: {subject}")
    return f"Email sent to {to_emails} with subject: {subject}"
```

- Now save the team and run the team
Here is an input to provide

```
I would like to get the latest arxiv papers from the last 7 days and send an email to xxx@test.com
```

- Make sure there is no error in the code
- If there is error, try to troubleshoot the code and fix it
- If there is no email coming, then check the error or the environment variables

![info](https://github.com/balakreshnan/Samples2025/blob/main/autogen/images/arxivpap-1.jpg 'RagChat')
![info](https://github.com/balakreshnan/Samples2025/blob/main/autogen/images/arxivpap-2.jpg 'RagChat')

- Try with multiple times to see if the email is coming
- Check your email inbox
- If you got the email then you are good to go.