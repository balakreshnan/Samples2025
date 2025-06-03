# Web Content and Email Summarization with Autogen using Autogen Studio

## Introduction

- This example demonstrates how to use Autogen to summarize web content and emails.
- Using Multimodal Web Surfer, we can extract and summarize information from various web pages.
- Ability to ask for various content and ability to provide email address to send the summary.
- This is a simple example to show how to use Autogen.
- This is not production-ready code.
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

![info](https://github.com/balakreshnan/Samples2025/blob/main/autogen/images/webcontentemail-1.jpg 'RagChat')

- Add web surfer system to the team
- configure the parameters for the web surfer system

![info](https://github.com/balakreshnan/Samples2025/blob/main/autogen/images/webcontentemail-2.jpg 'RagChat')

- Make sure Azure Open AI is configured in the Autogen Studio

![info](https://github.com/balakreshnan/Samples2025/blob/main/autogen/images/webcontentemail-3.jpg 'RagChat')

- Now we can create a Email Assistant system to send the summary of the web content

![info](https://github.com/balakreshnan/Samples2025/blob/main/autogen/images/webcontentemail-4.jpg 'RagChat')

![info](https://github.com/balakreshnan/Samples2025/blob/main/autogen/images/webcontentemail-5.jpg 'RagChat')

- Here is the code for the Email Assistant system

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

- now save the team and run the team
- Here is the question to ask the team

```
get the latest news for nvidia and email to aaaa@bbbbbbb.ccc
```

![info](https://github.com/balakreshnan/Samples2025/blob/main/autogen/images/webcontentemail-6.jpg 'RagChat')
![info](https://github.com/balakreshnan/Samples2025/blob/main/autogen/images/webcontentemail-7.jpg 'RagChat')

- Try with different prompts and see email is sent
- More experiments can be done with the team.