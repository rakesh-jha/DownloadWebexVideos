# DownloadWebexVideos
Download the WebEx videos using NBR API

1. Create a console application using visual studio

2. Add the below AppSetting in your App.config file, replace values as per your settings
    ```
    <add key="soapenv" value="http://schemas.xmlsoap.org/soap/envelope/" />
    <add key="xsd" value="http://www.w3.org/2001/XMLSchema" />
    <add key="xsi" value="http://ww.w3.org/2001/XMLSchema-instance" />
    <add key="encodingStyle" value="http://schemas.xmlsoap.org/soap/encoding/" />
    <add key="serv" value="http://www.webex.com/schemas/2002/06/service" />
    <add key="NBRAPIUrl" value="http://nsg1wss.webex.com/nbr/services/NBRStorageService" />
    <add key="XMLAPIUrl" value="https://xxxx.webex.com/WBXService/XMLService" />
    <add key="SiteId" value="xxxxxx" />
    <add key="SiteName" value="xxxx" />
    <add key="UserName" value="xxxxxx" />
    <add key="Password" value="xxxxxx" />
    ```
    
3. Create a new class named 'WebExXMLRequest.cs', this class will contain the XML requests for WebEx services
    ```
    using System.Configuration;
    namespace WebExDownloadVideo
    {
        public class WebExXmlRequest
        {
            string schema_soapenv { get; set; }
            string schema_xsd { get; set; }
            string schema_xsi { get; set; }
            string schema_encodingStyle { get; set; }
            string schema_serv { get; set; }
            string siteId { get; set; }
            string siteName { get; set; }
            string userName { get; set; }
            string password { get; set; }

            public WebExXmlRequest()
            {
                schema_soapenv = ConfigurationManager.AppSettings["soapenv"];
                schema_xsd = ConfigurationManager.AppSettings["xsd"];
                schema_xsi = ConfigurationManager.AppSettings["xsi"];
                schema_encodingStyle = ConfigurationManager.AppSettings["encodingStyle"];
                schema_serv = ConfigurationManager.AppSettings["serv"];
                siteId = ConfigurationManager.AppSettings["SiteId"];
                siteName = ConfigurationManager.AppSettings["SiteName"];
                userName = ConfigurationManager.AppSettings["UserName"];
                password = ConfigurationManager.AppSettings["Password"];
            }
            public string ConstructXMLForStorageAccessTicket()
            {
                return "<?xml version='1.0' encoding='UTF-8'?>\r\n" +
                        "<soapenv:Envelope xmlns:soapenv='" + schema_soapenv + "' xmlns:xsd='" + schema_xsd + "' xmlns:xsi='" + schema_xsi + "'>\r\n" +
                            "<soapenv:Body>\r\n" +
                                "<ns1:getStorageAccessTicket soapenv:encodingStyle='" + schema_encodingStyle + "' xmlns:ns1='NBRStorageService'>\r\n" +
                                    "<siteId xsi:type='xsd:long'>" + siteId + "</siteId>\r\n" +
                                    "<username xsi:type='xsd:string'>" + userName + "</username>\r\n" +
                                    "<password xsi:type='xsd:string'>" + password + "</password>\r\n" +
                                "</ns1:getStorageAccessTicket>\r\n" +
                            "</soapenv:Body>\r\n" +
                        "</soapenv:Envelope>";
            }

            public string ConstructXMLForGettingRecordId(string meetingKey)
            {
                return "<?xml version=\"1.0\" encoding=\"ISO-8859-1\"?>\r\n" +
                        "<serv:message xmlns:xsi=\"http://www.w3.org/2001/XMLSchema-instance\" xmlns:serv=\"http://www.webex.com/schemas/2002/06/service\">\r\n" +
                            "<header>\r\n" +
                                "<securityContext>\r\n" +
                                    "<webExID>" + userName + "</webExID>\r\n" +
                                    "<password>" + password + "</password>\r\n" +
                                    "<siteName>" + siteName + "</siteName>\r\n" +
                                "</securityContext>\r\n" +
                            "</header>\r\n" +
                            "<body>\r\n" +
                                "<bodyContent xsi:type=\"java:com.webex.service.binding.ep.LstRecording\">\r\n" +
                                    "<sessionKey>" + meetingKey + "</sessionKey>\r\n" +
                                    "<hostWebExID>" + userName + "</hostWebExID>\r\n" +
                                "</bodyContent>\r\n" +
                            "</body>\r\n" +
                        "</serv:message>";
            }

            public string ConstructXMLForDownloadingNBRFile(string recordId, string ticket)
            {
                return "<?xml version='1.0' encoding='UTF-8'?>\r\n" +
                        "<soapenv:Envelope xmlns:soapenv='" + schema_soapenv + "' xmlns:xsd='" + schema_xsd + "' xmlns:xsi='" + schema_xsi + "'>\r\n" +
                            "<soapenv:Body>\r\n" +
                                "<ns1:downloadNBRStorageFile soapenv:encodingStyle='" + schema_encodingStyle + "' xmlns:ns1=\"NBRStorageService\">\r\n" +
                                    "<siteId xsi:type='xsd:long'>" + siteId + "</siteId>\r\n" +
                                    "<recordId xsi:type=\"xsd:long\">" + recordId + "</recordId>\r\n" +
                                    "<ticket xsi:type=\"xsd:string\">" + ticket + "</ticket>\r\n" +
                                "</ns1:downloadNBRStorageFile>\r\n" +
                            "</soapenv:Body>\r\n" +
                        "</soapenv:Envelope>";
            }
        }
    }
    ```
4. Now, create another class named 'WebExHelper.cs', this class will send the requests to WebEx XML/NBR services
    ```
    using System;
    using System.Configuration;
    using System.IO;
    using System.Net;
    using System.Text;
    using System.Xml;

    namespace WebExDownloadVideo
    {
        public class WebExHelper
        {
            public string GetStorageAccessTicket()
            {
                var apiUrl = ConfigurationManager.AppSettings["NBRAPIUrl"];
                var xml = new WebExXmlRequest().ConstructXMLForStorageAccessTicket();
                var webRequest = System.Net.WebRequest.Create(apiUrl);
                webRequest.Headers.Add("SOAPAction", apiUrl + "/getStorageAccessTicket");
                webRequest.Method = System.Net.WebRequestMethods.Http.Post;
                webRequest.ContentType = "text/xml;charset=utf-8";

                var byteArray = Encoding.UTF8.GetBytes(xml.ToString());
                webRequest.ContentLength = byteArray.Length;

                var dataStream = webRequest.GetRequestStream();
                dataStream.Write(byteArray, 0, byteArray.Length);
                dataStream.Close();
                var response = webRequest.GetResponse();

                dataStream = response.GetResponseStream();
                var reader = new System.IO.StreamReader(dataStream);
                string rawResponse = reader.ReadToEnd();

                dataStream.Close();
                reader.Close();
                response.Close();

                string result = string.Empty;
                XmlDocument xmlResponse = new XmlDocument();
                xmlResponse.LoadXml(rawResponse);

                XmlNodeList ticket = xmlResponse.GetElementsByTagName("ns1:getStorageAccessTicketReturn");
                if (ticket.Count > 0)
                {
                    result = ticket[0].InnerText;
                }
                return result;
            }

            public string GetRecordId(string meetingKey)
            {
                var apiUrl = ConfigurationManager.AppSettings["XMLAPIUrl"];
                var xml = new WebExXmlRequest().ConstructXMLForGettingRecordId(meetingKey);
                var webRequest = WebRequest.Create(apiUrl);
                webRequest.Method = WebRequestMethods.Http.Post;
                webRequest.ContentType = "application/x-www-form-urlencoded";

                var byteArray = Encoding.UTF8.GetBytes(xml.ToString());
                webRequest.ContentLength = byteArray.Length;

                var dataStream = webRequest.GetRequestStream();
                dataStream.Write(byteArray, 0, byteArray.Length);
                dataStream.Close();
                var response = webRequest.GetResponse();

                dataStream = response.GetResponseStream();
                var reader = new System.IO.StreamReader(dataStream);
                string rawResponse = reader.ReadToEnd();

                dataStream.Close();
                reader.Close();
                response.Close();

                string result = string.Empty;
                XmlDocument xmlResponse = new XmlDocument();
                xmlResponse.LoadXml(rawResponse);

                XmlNodeList recordingIds = xmlResponse.GetElementsByTagName("ep:recordingID");
                if (recordingIds.Count > 0)
                {
                    result = recordingIds[0].InnerText;
                }
                return result;
            }

            public void DownloadNBRStorageFile(string ticket, string recordId, string meetingKey)
            {
                string filePath = "d:\\WebEx_Videos\\" + meetingKey + ".arf";
                var apiUrl = ConfigurationManager.AppSettings["NBRAPIUrl"];
                var xml = new WebExXmlRequest().ConstructXMLForDownloadingNBRFile(recordId, ticket);
                var webRequest = System.Net.WebRequest.Create(apiUrl);
                webRequest.Headers.Add("SOAPAction", apiUrl + "/downloadNBRStorageFile");
                webRequest.Method = System.Net.WebRequestMethods.Http.Post;
                webRequest.ContentType = "text/xml;charset=utf-8";

                var byteArray = Encoding.UTF8.GetBytes(xml.ToString());
                webRequest.ContentLength = byteArray.Length;

                var dataStream = webRequest.GetRequestStream();
                dataStream.Write(byteArray, 0, byteArray.Length);
                dataStream.Close();
                var response = webRequest.GetResponse();

                dataStream = response.GetResponseStream();
                StringBuilder sb = new StringBuilder();
                int bytetoread = 10;
                string str = "";
                using (FileStream fs = File.Create(filePath))
                {
                    Byte[] buffer = new Byte[1024];

                    int read = dataStream.Read(buffer, 0, buffer.Length);

                    str = Encoding.UTF8.GetString(buffer);

                    buffer = new Byte[200];
                    read = dataStream.Read(buffer, 0, buffer.Length);
                    str = Encoding.UTF8.GetString(buffer);

                    while (read > 0)
                    {
                        buffer = new Byte[bytetoread];
                        read = dataStream.Read(buffer, 0, buffer.Length);
                        str = Encoding.UTF8.GetString(buffer);
                        if (str.Contains("<"))
                        {
                            bytetoread = 1;
                        }
                        if (str.Contains(">"))
                        {
                            break;
                        }
                    }

                    int cnt = 4;
                    while (cnt > 0)
                    {
                        buffer = new Byte[1];
                        read = dataStream.Read(buffer, 0, buffer.Length);
                        str = Encoding.UTF8.GetString(buffer);
                        cnt--;
                    }

                    buffer = new Byte[1024];
                    read = dataStream.Read(buffer, 0, buffer.Length);
                    while (read > 0)
                    {
                        fs.Write(buffer, 0, read);
                        buffer = new Byte[1024];
                        read = dataStream.Read(buffer, 0, buffer.Length);
                    }
                }
                dataStream.Close();
            }
        }
    }
    ```
5. In your 'Program.cs' file, paste the code below
    ```
    using System;
    using System.Collections.Generic;
    using System.IO;
    using System.Linq;

    namespace WebExDownloadVideo
    {
        class Program
        {
            static void Main(string[] args)
            {
                try
                {
                    int totalDownloads = 1;
                    //To keep the appointment ids where record id does not exist
                    List<string> recordIdNotFound = new List<string>();
                    WebExHelper obj = new WebExHelper();

                    var meetingKeys = new List<string>() { "1234567", "8765444" }; //Make a request to your DB to fetch meeting key

                    foreach (var item in meetingKeys)
                    {
                        bool checkIfFileExist = File.Exists("d:\\WebEx_Videos\\" + item + ".arf");
                        if (checkIfFileExist)
                        {
                            Console.WriteLine("File already exist for meeting key #" + item);
                            totalDownloads++;
                        }
                        else
                        {
                            try
                            {
                                Console.WriteLine("=====================================================================");
                                #region Get-RecordId
                                Console.WriteLine("Fetching RecordId for meeting key #" + item + "..." + Environment.NewLine);
                                var recordId = obj.GetRecordId(item);
                                Console.WriteLine("Record ID: " + recordId + Environment.NewLine);
                                #endregion

                                if (!string.IsNullOrEmpty(recordId))
                                {
                                    #region Get-Access-Ticket
                                    Console.WriteLine("Fetching ticket for accessing NBR storage server..." + Environment.NewLine);
                                    var ticket = obj.GetStorageAccessTicket();
                                    Console.WriteLine("Ticket : " + ticket + Environment.NewLine);
                                    #endregion

                                    if (!string.IsNullOrEmpty(ticket))
                                    {
                                        #region Download-Recording
                                        Console.WriteLine("Downloading file for meeting key # : " + item + "..." + Environment.NewLine);
                                        obj.DownloadNBRStorageFile(ticket, recordId, item);
                                        Console.WriteLine("File downloaded successfully for meeting key #: " + item + Environment.NewLine);
                                        Console.WriteLine("Total files downloaded : " + totalDownloads + Environment.NewLine);
                                        totalDownloads++;
                                        #endregion 
                                    }
                                }
                                else
                                {
                                    recordIdNotFound.Add(item);
                                }
                                Console.WriteLine("=====================================================================");
                            }
                            catch (Exception ex)
                            {
                                Console.WriteLine("Error downloading the file for appointment id : " + item + Environment.NewLine + ex.Message);
                            }
                        }
                    }

                    if (recordIdNotFound != null && recordIdNotFound.Any())
                    {
                        Console.WriteLine("RecordId not found for below Appointments : " + Environment.NewLine);
                        foreach (var item in recordIdNotFound)
                        {
                            Console.WriteLine(item);
                        }
                    }

                    Console.Read();
                }
                catch (Exception ex)
                {
                    Console.WriteLine(ex.Message);
                    Console.ReadLine();
                }
            }
        }
    }
    ``` 
6. Build and run the application.



