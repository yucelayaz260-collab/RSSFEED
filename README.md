# RSSFEED
I’m building an RSS reader just as a hobby. I left it at the alpha stage. If I can, I’ll share the code; I wrote it in C#. code below (ı am using google Translate)

using System;
using System.Collections.Generic;
using System.Drawing;
using System.IO;
using System.Linq;
using System.Net.Http;
using System.ServiceModel.Syndication;
using System.Threading.Tasks;
using System.Windows.Forms;
using System.Xml;
using Microsoft.Data.Sqlite;
using Microsoft.Web.WebView2.WinForms;

namespace RSSFEED
{
    public partial class Form1 : Form
    {
        private TextBox txtUrl;
        private Button btnAdd;
        private Button btnRefresh;
        private ListBox lstFeeds;
        private ListBox lstNews;
        private WebView2 browser;
        private NotifyIcon notifier;
        private System.Windows.Forms.Timer timer;

        private readonly string dbPath;
        private readonly string connectionString;

        private readonly List<Feed> feeds = new();
        private readonly Dictionary<string, List<Article>> articlesByFeed = new();

        private bool checking = false;

        private static readonly HttpClient http = CreateHttpClient();

        private static HttpClient CreateHttpClient()
        {
            var client = new HttpClient();
            client.Timeout = TimeSpan.FromSeconds(25);
            client.DefaultRequestHeaders.UserAgent.ParseAdd(
                "RSSFEED/1.0 (Windows RSS Reader)");
            client.DefaultRequestHeaders.Accept.ParseAdd(
                "application/rss+xml, application/atom+xml, application/xml, text/xml");
            return client;
        }

        public Form1()
        {
            InitializeComponent();

            dbPath = Path.Combine(
                Application.LocalUserAppDataPath, "rssfeed.db");

            connectionString = $"Data Source={dbPath}";

            BuildInterface();
        }

        private void BuildInterface()
        {
            Text = "RSSFEED";
            Size = new Size(1250, 780);
            MinimumSize = new Size(950, 600);
            StartPosition = FormStartPosition.CenterScreen;
            BackColor = Color.FromArgb(245, 247, 250);

            Label title = new Label
            {
                Text = "RSSFEED",
                Font = new Font("Segoe UI", 23, FontStyle.Bold),
                Location = new Point(20, 15),
                AutoSize = true
            };

            txtUrl = new TextBox
            {
                Location = new Point(25, 75),
                Size = new Size(750, 35),
                Font = new Font("Segoe UI", 11),
                PlaceholderText = "RSS adresini gir..."
            };

            btnAdd = new Button
            {
                Text = "RSS Ekle",
                Location = new Point(790, 73),
                Size = new Size(110, 35)
            };

            btnRefresh = new Button
            {
                Text = "Şimdi Kontrol Et",
                Location = new Point(910, 73),
                Size = new Size(150, 35)
            };

            Label feedsLabel = new Label
            {
                Text = "KAYNAKLAR",
                Location = new Point(25, 125),
                AutoSize = true,
                Font = new Font("Segoe UI", 10, FontStyle.Bold)
            };

            Label newsLabel = new Label
            {
                Text = "HABERLER",
                Location = new Point(265, 125),
                AutoSize = true,
                Font = new Font("Segoe UI", 10, FontStyle.Bold)
            };

            Label articleLabel = new Label
            {
                Text = "HABER OKUYUCU",
                Location = new Point(630, 125),
                AutoSize = true,
                Font = new Font("Segoe UI", 10, FontStyle.Bold)
            };

            lstFeeds = new ListBox
            {
                Location = new Point(25, 155),
                Size = new Size(220, 530),
                Anchor = AnchorStyles.Top | AnchorStyles.Bottom | AnchorStyles.Left
            };

            lstNews = new ListBox
            {
                Location = new Point(265, 155),
                Size = new Size(350, 530),
                Font = new Font("Segoe UI", 10),
                Anchor = AnchorStyles.Top | AnchorStyles.Bottom | AnchorStyles.Left
            };

            browser = new WebView2
            {
                Location = new Point(630, 155),
                Size = new Size(570, 530),
                Anchor = AnchorStyles.Top | AnchorStyles.Bottom |
                         AnchorStyles.Left | AnchorStyles.Right
            };

            Controls.AddRange(new Control[]
            {
                title, txtUrl, btnAdd, btnRefresh,
                feedsLabel, newsLabel, articleLabel,
                lstFeeds, lstNews, browser
            });

            btnAdd.Click += BtnAdd_Click;
            btnRefresh.Click += BtnRefresh_Click;
            lstFeeds.SelectedIndexChanged += LstFeeds_SelectedIndexChanged;
            lstNews.SelectedIndexChanged += LstNews_SelectedIndexChanged;

            notifier = new NotifyIcon
            {
                Icon = SystemIcons.Information,
                Visible = true,
                Text = "RSSFEED"
            };

            timer = new System.Windows.Forms.Timer
            {
                Interval = 5 * 60 * 1000
            };

            timer.Tick += async (s, e) => await CheckAllFeedsAsync(true);
        }

        private async void Form1_Load(object sender, EventArgs e)
        {
            InitializeDatabase();
            LoadSavedFeeds();

            try
            {
                await browser.EnsureCoreWebView2Async();
            }
            catch
            {
                MessageBox.Show(
                    "WebView2 başlatılamadı. WebView2 Runtime kurulu mu kontrol et.");
            }

            await CheckAllFeedsAsync(true);
            timer.Start();
        }

        private void InitializeDatabase()
        {
            Directory.CreateDirectory(
                Path.GetDirectoryName(dbPath)!);

            using var connection = new SqliteConnection(connectionString);
            connection.Open();

            using var command = connection.CreateCommand();

            command.CommandText = @"
                CREATE TABLE IF NOT EXISTS Feeds (
                    Id INTEGER PRIMARY KEY AUTOINCREMENT,
                    Url TEXT NOT NULL UNIQUE,
                    Title TEXT NOT NULL
                );

                CREATE TABLE IF NOT EXISTS Seen (
                    FeedUrl TEXT NOT NULL,
                    ItemKey TEXT NOT NULL,
                    PRIMARY KEY (FeedUrl, ItemKey)
                );
            ";

            command.ExecuteNonQuery();
        }

        private void LoadSavedFeeds()
        {
            feeds.Clear();
            lstFeeds.Items.Clear();

            using var connection = new SqliteConnection(connectionString);
            connection.Open();

            using var command = connection.CreateCommand();
            command.CommandText = "SELECT Url, Title FROM Feeds";

            using var reader = command.ExecuteReader();

            while (reader.Read())
            {
                Feed feed = new Feed
                {
                    Url = reader.GetString(0),
                    Title = reader.GetString(1)
                };

                feeds.Add(feed);
                lstFeeds.Items.Add(feed);
            }
        }

        private async void BtnAdd_Click(object sender, EventArgs e)
        {
            string url = txtUrl.Text.Trim();

            if (!Uri.TryCreate(url, UriKind.Absolute, out Uri uri) ||
                (uri.Scheme != "http" && uri.Scheme != "https"))
            {
                MessageBox.Show("Geçerli bir RSS adresi gir!");
                return;
            }

            btnAdd.Enabled = false;

            try
            {
                var result = await DownloadFeedAsync(url);

                if (feeds.Any(f => f.Url == url))
                {
                    MessageBox.Show("Bu RSS kaynağı zaten eklenmiş.");
                    return;
                }

                if (result == null)
                {
                    MessageBox.Show("RSS kaynağı okunamadı.");
                    return;
                }

                Feed feed = new Feed
                {
                    Url = url,
                    Title = result.Title
                };

                using var connection = new SqliteConnection(connectionString);
                connection.Open();

                using var command = connection.CreateCommand();
                command.CommandText =
                    "INSERT INTO Feeds (Url, Title) VALUES ($url, $title)";

                command.Parameters.AddWithValue("$url", url);
                command.Parameters.AddWithValue("$title", feed.Title);
                command.ExecuteNonQuery();

                // İlk eklemede eski haberler bildirim oluşturmasın.
                foreach (var article in result.Articles)
                    MarkSeen(url, article.Key);

                feeds.Add(feed);
                lstFeeds.Items.Add(feed);

                articlesByFeed[url] = result.Articles;
                lstFeeds.SelectedItem = feed;

                txtUrl.Clear();

                MessageBox.Show("RSS kaynağı başarıyla eklendi!");
            }
            catch (Exception ex)
            {
                MessageBox.Show("RSS hatası:\n" + ex.Message);
            }
            finally
            {
                btnAdd.Enabled = true;
            }
        }

        private async Task<FeedResult> DownloadFeedAsync(string url)
        {
            using var response = await http.GetAsync(url);
            response.EnsureSuccessStatusCode();

            using var stream = await response.Content.ReadAsStreamAsync();

            var settings = new XmlReaderSettings
            {
                DtdProcessing = DtdProcessing.Prohibit,
                XmlResolver = null
            };

            using var reader = XmlReader.Create(stream, settings);

            var feed = SyndicationFeed.Load(reader);

            if (feed == null)
                return null;

            var result = new FeedResult
            {
                Title = feed.Title?.Text ?? url
            };

            foreach (var item in feed.Items)
            {
                string articleUrl =
                    item.Links.FirstOrDefault(
                        l => l.RelationshipType == "alternate")
                    ?.Uri?.ToString() ?? "";

                string articleTitle =
                    item.Title?.Text ?? "Başlıksız haber";

                string key = !string.IsNullOrWhiteSpace(item.Id)
                    ? item.Id
                    : !string.IsNullOrWhiteSpace(articleUrl)
                        ? articleUrl
                        : articleTitle + item.PublishDate;

                result.Articles.Add(new Article
                {
                    Title = articleTitle,
                    Url = articleUrl,
                    Key = key,
                    Date = item.PublishDate.LocalDateTime,
                    Summary = item.Summary?.Text ?? ""
                });
            }

            return result;
        }

        private async Task CheckAllFeedsAsync(bool notify)
        {
            if (checking)
                return;

            checking = true;

            int newCount = 0;
            string firstNewTitle = "";

            try
            {
                foreach (var feed in feeds.ToList())
                {
                    try
                    {
                        var result = await DownloadFeedAsync(feed.Url);

                        if (result == null)
                            continue;

                        feed.Title = result.Title;
                        UpdateFeedTitle(feed);

                        var articles = result.Articles;

                        articlesByFeed[feed.Url] = articles;

                        foreach (var article in articles)
                        {
                            if (!IsSeen(feed.Url, article.Key))
                            {
                                MarkSeen(feed.Url, article.Key);

                                newCount++;

                                if (string.IsNullOrEmpty(firstNewTitle))
                                    firstNewTitle = article.Title;
                            }
                        }
                    }
                    catch
                    {
                        // Bir kaynak hata verse bile diğerleri kontrol edilir.
                    }
                }

                RefreshNewsList();

                if (notify && newCount > 0)
                {
                    notifier.ShowBalloonTip(
                        5000,
                        $"RSSFEED - {newCount} yeni haber",
                        firstNewTitle,
                        ToolTipIcon.Info
                    );
                }
            }
            finally
            {
                checking = false;
            }
        }

        private bool IsSeen(string url, string key)
        {
            using var connection = new SqliteConnection(connectionString);
            connection.Open();

            using var command = connection.CreateCommand();

            command.CommandText =
                "SELECT COUNT(*) FROM Seen WHERE FeedUrl=$url AND ItemKey=$key";

            command.Parameters.AddWithValue("$url", url);
            command.Parameters.AddWithValue("$key", key);

            return Convert.ToInt32(command.ExecuteScalar()) > 0;
        }

        private void MarkSeen(string url, string key)
        {
            using var connection = new SqliteConnection(connectionString);
            connection.Open();

            using var command = connection.CreateCommand();

            command.CommandText =
                "INSERT OR IGNORE INTO Seen (FeedUrl, ItemKey) VALUES ($url, $key)";

            command.Parameters.AddWithValue("$url", url);
            command.Parameters.AddWithValue("$key", key);

            command.ExecuteNonQuery();
        }

        private void UpdateFeedTitle(Feed feed)
        {
            using var connection = new SqliteConnection(connectionString);
            connection.Open();

            using var command = connection.CreateCommand();

            command.CommandText =
                "UPDATE Feeds SET Title=$title WHERE Url=$url";

            command.Parameters.AddWithValue("$title", feed.Title);
            command.Parameters.AddWithValue("$url", feed.Url);

            command.ExecuteNonQuery();
        }

        private void RefreshNewsList()
        {
            if (lstFeeds.SelectedItem is not Feed feed)
                return;

            lstNews.Items.Clear();

            if (!articlesByFeed.TryGetValue(feed.Url, out var articles))
                return;

            foreach (var article in articles)
                lstNews.Items.Add(article);
        }

        private void LstFeeds_SelectedIndexChanged(object sender, EventArgs e)
        {
            RefreshNewsList();
        }

        private async void LstNews_SelectedIndexChanged(object sender, EventArgs e)
        {
            if (lstNews.SelectedItem is not Article article)
                return;

            if (!string.IsNullOrWhiteSpace(article.Url) &&
                Uri.TryCreate(article.Url, UriKind.Absolute, out Uri uri) &&
                (uri.Scheme == "http" || uri.Scheme == "https"))
            {
                try
                {
                    await browser.EnsureCoreWebView2Async();
                    browser.CoreWebView2.Navigate(article.Url);
                }
                catch (Exception ex)
                {
                    MessageBox.Show("Haber açılamadı:\n" + ex.Message);
                }
            }
            else
            {
                MessageBox.Show(
                    article.Title + "\n\n" + article.Summary,
                    "RSSFEED - Haber Özeti");
            }
        }

        private async void BtnRefresh_Click(object sender, EventArgs e)
        {
            btnRefresh.Enabled = false;

            try
            {
                await CheckAllFeedsAsync(true);
            }
            finally
            {
                btnRefresh.Enabled = true;
            }
        }

        protected override void OnFormClosing(FormClosingEventArgs e)
        {
            timer?.Stop();

            if (notifier != null)
            {
                notifier.Visible = false;
                notifier.Dispose();
            }

            base.OnFormClosing(e);
        }
    }

    public class Feed
    {
        public string Url { get; set; }
        public string Title { get; set; }

        public override string ToString() => Title;
    }

    public class Article
    {
        public string Title { get; set; }
        public string Url { get; set; }
        public string Key { get; set; }
        public string Summary { get; set; }
        public DateTime Date { get; set; }

        public override string ToString()
        {
            return $"{Date:dd.MM HH:mm} | {Title}";
        }
    }

    public class FeedResult
    {
        public string Title { get; set; }
        public List<Article> Articles { get; set; } = new();
    }
}

