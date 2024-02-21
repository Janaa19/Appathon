import SwiftUI

struct ChatView: View {
    @State private var messages: [Message] = []
    @State private var newMessage: String = ""
    @State private var webSocketTask: URLSessionWebSocketTask!

    var requestID: UUID
    var userName: String

    var body: some View {
        GeometryReader { geometry in
            ScrollViewReader { scrollView in
                ZStack {
                    Image("osu_O")
                        .resizable()
                        .opacity(0.1)
                        .aspectRatio(contentMode: .fit)
                        .frame(maxWidth: .infinity)
                        .padding(.bottom, 2)
                    VStack {
                        ScrollView {
                            ForEach(messages, id: \.id) { message in
                                if message.userName == userName {
                                    // Sender's message (on the right)
                                    HStack {
                                        Spacer()
                                        Spacer()
                                        VStack(alignment: .trailing, spacing: 2.5) {
                                            Text(message.message)
                                                .padding(10)
                                                .background(Color(red: 186/255, green: 12/255, blue: 47/255))
                                                .foregroundColor(.white)
                                                .clipShape(RoundedRectangle(cornerRadius: 10))
                                            Text(message.userName)
                                                .font(.caption)
                                                .foregroundColor(.gray)
                                        }
                                        .padding(.trailing, 5) // Adjusted spacing from the right edge
                                    }
                                } else {
                                    // Receiver's message (on the left)
                                    HStack {
                                        VStack(alignment: .leading, spacing: 2.5) {
                                            Text(message.userName)
                                                .font(.caption)
                                                .foregroundColor(.gray)
                                            Text(message.message)
                                                .padding(10)
                                                .background(Color(red: 77/255, green: 77/255, blue: 77/255)) // Adjust colors as needed
                                                .foregroundColor(.white)
                                                .clipShape(RoundedRectangle(cornerRadius: 10))
                                        }
                                        Spacer()
                                            .padding(.leading, 5) // Adjusted spacing from the left edge
                                    }
                                }
                            }
                        }
                        .padding(.horizontal, 20) // Adjusted spacing from both horizontal edges
                        .padding(.bottom, 60)
                    }
                    .padding(.vertical, 8) // Adjusted spacing from both vertical edges
                    .onAppear {
                        connectWebSocket()
                        getChatMessages()
                    }
                    
                    VStack {
                        Spacer()
                        HStack {
                            TextField("Type a message", text: $newMessage)
                                .textFieldStyle(RoundedBorderTextFieldStyle())
                            Button(action: sendMessage) {
                                Text("Send")
                                    .foregroundColor(Color(red: 186/255, green: 12/255, blue: 47/255))
                            }
                        }
                        .padding()
                    }
                }
            }
        }
    }
        private func sendMessage() {
            guard !newMessage.isEmpty else { return }

            // Append the message first, assuming optimistic UI update
            messages.append(Message(id: UUID().uuidString, sentAt: Date(), message: newMessage, userName: userName, requestID: requestID.uuidString))

            // create message
            let message = URLSessionWebSocketTask.Message.string(newMessage)
            // send message
            webSocketTask.send(message) { error in
                if let error = error {
                    // failed
                    print("Error sending data: \(error)")
                }
            }
            newMessage = ""
            print("Sent data to WS!")
        }

        private func connectWebSocket() {
            let wsURLStr = "\(LinkToDatabase.wslink)/\(requestID)"
            guard let wsURL = URL(string: wsURLStr) else {
                print("Invalid WebSocket URL")
                return
            }

            var request = URLRequest(url: wsURL)
            request.setValue(userName, forHTTPHeaderField: "userName")

            webSocketTask = URLSession.shared.webSocketTask(with: request)
            webSocketTask.resume()
            print("WebSocket connected")

            receivedFromWS()
        }

        private func receivedFromWS() {
            webSocketTask.receive { [self] result in
                switch result {
                case .failure(let error):
                    print("Failed to receive message: \(error)")
                case .success(let message):
                    DispatchQueue.main.async {
                        switch message {
                        case .string(let str):
                            if let separatorIndex = str.firstIndex(of: ":") {
                                let userName = String(str.prefix(upTo: separatorIndex)).trimmingCharacters(in: .whitespaces)
                                let message = String(str.suffix(from: str.index(after: separatorIndex))).trimmingCharacters(in: .whitespaces)
                                let receivedMessage = Message(id: UUID().uuidString, sentAt: Date(), message: message, userName: userName, requestID: requestID.uuidString)
                                self.messages.append(receivedMessage)
                                print("Received string: \(str)")
                            } else {
                                print("Invalid message format: \(str)")
                            }
                        case .data(let data):
                            // Assuming the received data is text and converting it to string
                            if let text = String(data: data, encoding: .utf8) {
                                if let separatorIndex = text.firstIndex(of: ":") {
                                    let userName = String(text.prefix(upTo: separatorIndex)).trimmingCharacters(in: .whitespaces)
                                    let message = String(text.suffix(from: text.index(after: separatorIndex))).trimmingCharacters(in: .whitespaces)
                                    let receivedMessage = Message(id: UUID().uuidString, sentAt: Date(), message: message, userName: userName, requestID: requestID.uuidString)
                                    self.messages.append(receivedMessage)
                                    print("Received binary (as text): \(text)")
                                } else {
                                    print("Invalid message format: \(text)")
                                }
                            } else {
                                print("Received binary data: \(data)")
                            }
                        @unknown default:
                            print("Unknown data type received from WebSocket.")
                        }
                    }

                    // Recursive call to keep receiving messages
                    self.receivedFromWS()
                }
            }
        }

        private func getChatMessages() {
            let urlStr = "\(LinkToDatabase.link)/chats/\(requestID)"
            guard let url = URL(string: urlStr) else {
                print("Invalid API URL")
                return
            }

            var request = URLRequest(url: url)
            request.httpMethod = "GET"

            let decoder = JSONDecoder()
            decoder.dateDecodingStrategy = .iso8601 // Set the date decoding strategy

            URLSession.shared.dataTask(with: request) { data, response, error in
                if let data = data {
                    do {
                        let decodedResponse = try decoder.decode([Message].self, from: data)
                        // Sort the messages by the `sentAt` date
                        let sortedMessages = decodedResponse.sorted(by: { $0.sentAt < $1.sentAt })

                        DispatchQueue.main.async {
                            // update our UI with sortedMessages
                            messages.removeAll() // Assuming you want to replace existing messages
                            for message in sortedMessages {
                                messages.append(message)
                            }
                            // Here, you would also update any UI elements (e.g., UITableView) to reflect the new messages
                        }
                    } catch {
                        print("Decoding failed: \(error)")
                    }
                } else {
                    print("Fetch failed: \(error?.localizedDescription ?? "Unknown error")")
                }
            }.resume()
        }

    }

    struct ChatView_Previews: PreviewProvider {
        static var previews: some View {
            ChatView(requestID: UUID(), userName: "John")
        }
    }

    struct Message: Decodable, Identifiable {
        let id: String
        let sentAt: Date
        let message: String
        let userName: String
        let requestID: String
    }


