# Claude CLI Skills: Complete Implementation Guide

> **Document Version:** 1.0  
> **Purpose:** Step-by-step instructions for building reusable API-to-PDF-to-Email skills for Claude Code CLI in both **Python** and **C# (.NET)**.  
> **Save this file as:** `claude-cli-skills-guide.md`

---

## Table of Contents
1. [Overview: What is a Claude Skill?](#overview-what-is-a-claude-skill)
2. [Part II: C# Implementation](#part-ii-c-implementation)
4. [Using Skills Inside Claude Code](#using-skills-inside-claude-code)
5. [Summary & Comparison](#summary--comparison)

---

## Overview: What is a Claude Skill?

**Claude Code** is Anthropic’s agentic command-line interface. Unlike rigid plugin ecosystems, Claude Code uses autonomous **tools** (Bash, View, Edit, Glob, etc.) to read files, run commands, and write code.

A **"Skill"** is a reusable, automated workflow. The most robust way to build one is as a **standalone CLI program** that accepts runtime arguments. This guide teaches you how to build a skill that:

1. Calls an HTTP REST API.
2. Generates a PDF report from the JSON response.
3. Emails that PDF to a dynamic list of recipients provided at runtime.

### Example Runtime Syntax
```bash
pdf-skill http://restapi.com/getAllRevenueDataForThisWeek email1@test.com email2@test.com
```

## Part II: C# Implementation
### Prerequisites
* .NET 8.0 SDK (or newer) installed.
* Linux only: libfontconfig1 is required for PDF rendering.
  * `sudo apt-get update && sudo apt-get install -y libfontconfig1`
 
### Step 1 — Create the Skill Directory & Project
```
mkdir -p ~/.claude-skills/pdf-skill
cd ~/.claude-skills/pdf-skill

# Scaffold a console application
dotnet new console

dotnet add package QuestPDF
dotnet add package MailKit
```
### Step 2 — Write the Skill Code
Replace the contents of `Program.cs` with the following:
```csharp
using System.Text.Json;
using QuestPDF.Fluent;
using QuestPDF.Helpers;
using QuestPDF.Infrastructure;
using MailKit.Net.Smtp;
using MimeKit;

// Suppress the QuestPDF community license banner
QuestPDF.Settings.License = LicenseType.Community;

class Program
{
    static async Task<int> Main(string[] args)
    {
        if (args.Length < 2)
        {
            Console.WriteLine("Usage: pdf-skill <api_url> <email1> [email2] ...");
            return 1;
        }

        string apiUrl = args[0];
        string[] recipients = args[1..];
        string pdfPath = Path.Combine(
            Path.GetTempPath(),
            $"claude_revenue_report_{Guid.NewGuid()}.pdf"
        );

        // 1. Fetch data from REST API
        using var httpClient = new HttpClient();
        Console.WriteLine($"📡 Fetching data from: {apiUrl}");

        string rawJson;
        try
        {
            var response = await httpClient.GetAsync(apiUrl);
            response.EnsureSuccessStatusCode();
            rawJson = await response.Content.ReadAsStringAsync();
        }
        catch (Exception ex)
        {
            Console.WriteLine($"❌ Failed to fetch API data: {ex.Message}");
            return 1;
        }

        // 2. Generate PDF with QuestPDF
        Console.WriteLine($"📄 Generating PDF at: {pdfPath}");
        try
        {
            var doc = JsonDocument.Parse(rawJson);
            string prettyJson = JsonSerializer.Serialize(doc, new JsonSerializerOptions { WriteIndented = true });

            Document.Create(container =>
            {
                container.Page(page =>
                {
                    page.Size(PageSizes.A4);
                    page.Margin(2, Unit.Centimetre);
                    page.PageColor(Colors.White);
                    page.DefaultTextStyle(x => x.FontSize(10));

                    page.Header()
                        .Text("Weekly Revenue Report")
                        .SemiBold().FontSize(20).FontColor(Colors.Blue.Darken2).AlignCenter();

                    page.Content().PaddingVertical(1, Unit.Centimetre).Column(column =>
                    {
                        column.Item().Text($"Generated: {DateTime.Now:yyyy-MM-dd HH:mm}")
                              .FontSize(9).Italic().AlignCenter();

                        column.Item().PaddingVertical(0.5, Unit.Centimetre);

                        column.Item().Background(Colors.Grey.Lighten4)
                              .Padding(1, Unit.Centimetre)
                              .Text(text =>
                              {
                                  text.Line("API Response").SemiBold().FontSize(12);
                                  text.EmptyLine();
                                  text.Line(prettyJson).FontFamily("Courier New").FontSize(9);
                              });
                    });

                    page.Footer().AlignCenter().Text(text =>
                    {
                        text.Span("Page ").FontSize(9);
                        text.CurrentPageNumber().FontSize(9);
                        text.Span(" of ").FontSize(9);
                        text.TotalPages().FontSize(9);
                    });
                });
            }).GeneratePdf(pdfPath);
        }
        catch (Exception ex)
        {
            Console.WriteLine($"❌ Failed to generate PDF: {ex.Message}");
            return 1;
        }

        // 3. Email the PDF via MailKit
        Console.WriteLine($"📧 Sending email to: {string.Join(", ", recipients)}");

        string? smtpServer = Environment.GetEnvironmentVariable("SMTP_SERVER") ?? "smtp.gmail.com";
        string? smtpPortStr = Environment.GetEnvironmentVariable("SMTP_PORT") ?? "587";
        string? senderEmail = Environment.GetEnvironmentVariable("SMTP_USER");
        string? senderPassword = Environment.GetEnvironmentVariable("SMTP_PASS");

        if (!int.TryParse(smtpPortStr, out int smtpPort) ||
            string.IsNullOrWhiteSpace(senderEmail) ||
            string.IsNullOrWhiteSpace(senderPassword))
        {
            Console.WriteLine("❌ Error: Set SMTP_USER and SMTP_PASS environment variables.");
            Console.WriteLine("   Optionally set SMTP_SERVER and SMTP_PORT.");
            return 1;
        }

        var message = new MimeMessage();
        message.From.Add(new MailboxAddress("Claude PDF Skill", senderEmail));
        foreach (var r in recipients)
            message.To.Add(MailboxAddress.Parse(r));

        message.Subject = $"Revenue Report - {DateTime.Now:yyyy-MM-dd}";

        var builder = new BodyBuilder
        {
            TextBody = "Please find the attached weekly revenue report.\n\nGenerated by Claude PDF Skill."
        };
        builder.Attachments.Add(pdfPath);
        message.Body = builder.ToMessageBody();

        using var smtpClient = new SmtpClient();
        try
        {
            await smtpClient.ConnectAsync(smtpServer, smtpPort, MailKit.Security.SecureSocketOptions.StartTls);
            await smtpClient.AuthenticateAsync(senderEmail, senderPassword);
            await smtpClient.SendAsync(message);
            await smtpClient.DisconnectAsync(true);
            Console.WriteLine("✅ Email sent successfully.");
        }
        catch (Exception ex)
        {
            Console.WriteLine($"❌ Failed to send email: {ex.Message}");
            return 1;
        }

        Console.WriteLine($"\n🎉 Skill completed. Report delivered to {recipients.Length} recipient(s).");
        return 0;
    }
}
```
### Step 3 — Publish the Executable
```
dotnet publish -c Release -o ./publish

# On Linux / macOS, make it executable
chmod +x ./publish/pdf-skill
```
>**Windows Note:** The publish folder will contain **pdf-skill.exe**. Add that folder to your **System PATH** or reference it directly.

### Step 4 — Add Shell Alias
Add to your shell profile (`~/.zshrc`, `~/.bashrc`, etc.):

#### Option A: Standard command (`pdf-skill`)
Linux
```bash
alias pdf-skill='/home/YOUR_USERNAME/.claude-skills/pdf-skill/publish/pdf-skill'
```


#### Option B: Slash command (`/pdf-skill`)
```
/pdf-skill() {
    /home/YOUR_USERNAME/.claude-skills/pdf-skill/publish/pdf-skill "$@"
}
```
Reload your shell
```
source ~/.zshrc
```
#### Step 5 — Configure SMTP Credentials
```bash
export SMTP_USER="your-email@gmail.com"
export SMTP_PASS="your-app-specific-password"
export SMTP_SERVER="smtp.gmail.com"
export SMTP_PORT="587"
```
>**Gmail Users: Generate an App Password at myaccount.google.com/apppasswords.**
Reload:
```
source ~/.zshrc
```
#### Step 6 — Use the Skill
```
pdf-skill http://restapi.com/getAllRevenueDataForThisWeek email1@test.com email2@test.com
```
Or, if you chose Option B:
```
/pdf-skill http://restapi.com/getAllRevenueDataForThisWeek email1@test.com email2@test.com
```
Expected Output:
```
📡 Fetching data from: http://restapi.com/getAllRevenueDataForThisWeek
📄 Generating PDF at: /tmp/claude_revenue_report_a1b2c3d4.pdf
📧 Sending email to: email1@test.com, email2@test.com
✅ Email sent successfully.

🎉 Skill completed. Report delivered to 2 recipient(s).
```
