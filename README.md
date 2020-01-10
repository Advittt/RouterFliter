server:

```csharp
using System;
using System.Collections.Generic;
using System.Net;
using System.Net.Sockets;
using System.Text;


namespace multiServer
{

    class Program
    {

        private static readonly Socket serverSocket = new Socket(AddressFamily.InterNetwork, SocketType.Stream, ProtocolType.Tcp);
        private static readonly List<Socket> clientSockets = new List<Socket>();
        private const int BUFFER_SIZE = 2048;
        private const int PORT = 100;
        private static readonly byte[] buffer = new byte[BUFFER_SIZE];

        static void Main()
        {

            Console.Title = "Server";
            SetupServer();
            Console.ReadLine(); // When you press enter close everything
            CloseAllSockets();

        }



        private static void SetupServer()
        {

            Console.WriteLine("Setting up server...");
            serverSocket.Bind(new IPEndPoint(IPAddress.Any, PORT));
            serverSocket.Listen(0);
            serverSocket.BeginAccept(AcceptCallback, null);
            Console.WriteLine("Server setup complete");

        }



        /// <summary>
        /// Close all connected client (we do not need to shutdown the server socket as its connections
        /// are already closed with the clients).
        /// </summary>

        private static void CloseAllSockets()
        {

            foreach (Socket socket in clientSockets)
            {

                socket.Shutdown(SocketShutdown.Both);
                socket.Close();

            }
            serverSocket.Close();
        }

        private static void AcceptCallback(IAsyncResult AR)
        {
            Socket socket;
            try
            {

                socket = serverSocket.EndAccept(AR);

            }

            catch (ObjectDisposedException) 
            {

                return;

            }

            clientSockets.Add(socket);
            socket.BeginReceive(buffer, 0, BUFFER_SIZE, SocketFlags.None, ReceiveCallback, socket);
            Console.WriteLine("Client connected, waiting for request...");
            serverSocket.BeginAccept(AcceptCallback, null);

        }



        private static void ReceiveCallback(IAsyncResult AR)
        {

            Socket current = (Socket)AR.AsyncState;
            int received;

            try
            {

                received = current.EndReceive(AR);

            }

            catch (SocketException)

            {

                Console.WriteLine("Client forcefully disconnected");
                // Don't shutdown because the socket may be disposed and its disconnected anyway.
                current.Close();
                clientSockets.Remove(current);
                return;

            }



            byte[] recBuf = new byte[received];
            Array.Copy(buffer, recBuf, received);
            string text = Encoding.ASCII.GetString(recBuf);
            Console.WriteLine("Received Text: " + text);



            if (text.ToLower() == "get time") // Client requested time
            {

                Console.WriteLine("Text is a get time request");
                byte[] data = Encoding.ASCII.GetBytes(DateTime.Now.ToLongTimeString());
                current.Send(data);
                Console.WriteLine("Time sent to client");

            }

            else if (text.ToLower() == "exit") // Client wants to exit gracefully
            {

                // Always Shutdown before closing
                current.Shutdown(SocketShutdown.Both);
                current.Close();
                clientSockets.Remove(current);
                Console.WriteLine("Client disconnected");
                return;

            }

            else
            {

                Console.WriteLine("Text is an invalid request");
                byte[] data = Encoding.ASCII.GetBytes("Invalid request");
                current.Send(data);
                Console.WriteLine("Warning Sent");

            }
            current.BeginReceive(buffer, 0, BUFFER_SIZE, SocketFlags.None, ReceiveCallback, current);

        }

    }

}
```

client:

```csharp
using System;
using System.Net.Sockets;
using System.Net;
using System.Text;

namespace multiClientServer
{

    class Program
    {

        private static readonly Socket ClientSocket = new Socket
            (AddressFamily.InterNetwork, SocketType.Stream, ProtocolType.Tcp);
               
        private const int PORT = 100;
               
        static void Main()
        {
            Console.Title = "Client";
            ConnectToServer();
            RequestLoop();
            Exit();

        }

        private static void ConnectToServer()
        {

            int attempts = 0;
            while (!ClientSocket.Connected)
            {

                try
                {

                    attempts++;
                    Console.WriteLine("Connection attempt " + attempts);
                    // Change IPAddress.Loopback to a remote IP to connect to a remote host.
                    ClientSocket.Connect(IPAddress.Loopback, PORT);

                }

                catch (SocketException)
                {

                    Console.Clear();

                }

            }
            Console.Clear();
            Console.WriteLine("Connected");

        }

        private static void RequestLoop()
        {

            Console.WriteLine(@"<Type ""exit"" to properly disconnect client>");
            while (true)
            {

                SendRequest();
                ReceiveResponse();

            }

        }



        /// <summary>
        /// Close socket and exit program.
        /// </summary>

        private static void Exit()
        {

            SendString("exit"); // Tell the server we are exiting
            ClientSocket.Shutdown(SocketShutdown.Both);
            ClientSocket.Close();
            Environment.Exit(0);

        }



        private static void SendRequest()
        {

            Console.Write("Send a request: ");
            string request = Console.ReadLine();
            SendString(request);



            if (request.ToLower() == "exit")
            {

                Exit();

            }

        }



        /// <summary>
        /// Sends a string to the server with ASCII encoding.
        /// </summary>

        private static void SendString(string text)
        {

            byte[] buffer = Encoding.ASCII.GetBytes(text);
            ClientSocket.Send(buffer, 0, buffer.Length, SocketFlags.None);

        }



        private static void ReceiveResponse()
        {

            var buffer = new byte[2048];
            int received = ClientSocket.Receive(buffer, SocketFlags.None);
            if (received == 0) return;
            var data = new byte[received];
            Array.Copy(buffer, data, received);
            string text = Encoding.ASCII.GetString(data);
            Console.WriteLine(text);

        }

    }

}

```

