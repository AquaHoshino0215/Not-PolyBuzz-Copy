import React, { useState, useEffect, useRef } from 'react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from 'firebase/auth';
import { getFirestore, doc, setDoc, getDoc, collection, query, onSnapshot, orderBy, serverTimestamp } from 'firebase/firestore';

// Global variables provided by the Canvas environment
const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {};
const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? initialAuthToken : null;

// Componente para o indicador de digitação (Tween)
const TypingIndicator = () => {
  const [dots, setDots] = useState(1);

  useEffect(() => {
    const interval = setInterval(() => {
      setDots((prevDots) => (prevDots % 3) + 1);
    }, 300); // Altera os pontos a cada 300ms
    return () => clearInterval(interval); // Limpa o intervalo ao desmontar o componente
  }, []);

  return (
    <span>
      {'A IA está a escrever' + '.'.repeat(dots)}
    </span>
  );
};

function App() {
  const [chatHistory, setChatHistory] = useState([]);
  const [userInput, setUserInput] = useState('');
  const [currentCharacter, setCurrentCharacter] = useState(null);
  const [characterNameInput, setCharacterNameInput] = useState('');
  const [characterDescriptionInput, setCharacterDescriptionInput] = useState('');
  // State for file objects and their previews
  const [characterImageFile, setCharacterImageFile] = useState(null);
  const [characterImagePreview, setCharacterImagePreview] = useState('');
  const [backgroundImageFile, setBackgroundImageFile] = useState(null);
  const [backgroundImagePreview, setBackgroundImagePreview] = useState('');

  const [isLoading, setIsLoading] = useState(false);
  const [db, setDb] = useState(null);
  const [auth, setAuth] = useState(null);
  const [userId, setUserId] = useState(null);
  const [isAuthReady, setIsAuthReady] = useState(false);
  const chatContainerRef = useRef(null);

  // Initialize Firebase and handle authentication
  useEffect(() => {
    const initializeFirebase = async () => {
      try {
        const app = initializeApp(firebaseConfig);
        const firestore = getFirestore(app);
        const firebaseAuth = getAuth(app);
        setDb(firestore);
        setAuth(firebaseAuth);

        // Listen for auth state changes
        const unsubscribe = onAuthStateChanged(firebaseAuth, async (user) => {
          if (user) {
            setUserId(user.uid);
          } else {
            // Sign in anonymously if no initial token or user logs out
            await signInAnonymously(firebaseAuth);
          }
          setIsAuthReady(true); // Mark auth as ready after initial check
        });

        // Use custom token if available for initial sign-in
        if (initialAuthToken) {
          await signInWithCustomToken(firebaseAuth, initialAuthToken);
        }

        return () => unsubscribe(); // Cleanup auth listener
      } catch (error) {
        console.error("Erro ao inicializar o Firebase ou autenticar:", error);
      }
    };

    initializeFirebase();
  }, []);

  // Scroll to bottom of chat when new messages arrive
  useEffect(() => {
    if (chatContainerRef.current) {
      chatContainerRef.current.scrollTop = chatContainerRef.current.scrollHeight;
    }
  }, [chatHistory]);

  // Function to send message to Gemini API
  const handleSendMessage = async () => {
    if (!userInput.trim() || isLoading) return;

    const userMessage = { role: 'user', text: userInput };
    setChatHistory((prev) => [...prev, userMessage]);
    setUserInput('');
    setIsLoading(true);

    try {
      let prompt = userInput;
      if (currentCharacter) {
        // Updated instruction for more descriptive, character-driven responses, but relatively brief
        prompt = `You are ${currentCharacter.name}, a character with the following description: "${currentCharacter.description}". Respond as ${currentCharacter.name}, including actions or descriptions in your reply. Keep your response relatively brief and engaging. User says: "${userInput}"`;
      }

      const payload = {
        contents: [{ role: 'user', parts: [{ text: prompt }] }],
      };

      const apiKey = ""; // API key is provided by Canvas runtime for gemini-2.0-flash
      const apiUrl = `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent?key=${apiKey}`;

      const response = await fetch(apiUrl, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload),
      });

      const result = await response.json();

      if (result.candidates && result.candidates.length > 0 &&
          result.candidates[0].content && result.candidates[0].content.parts &&
          result.candidates[0].content.parts.length > 0) {
        const aiText = result.candidates[0].content.parts[0].text;
        setChatHistory((prev) => [...prev, { role: 'ai', text: aiText }]);

        // Optional: Save chat to Firestore (example structure)
        if (db && userId && currentCharacter) {
          const chatDocRef = doc(db, `artifacts/${appId}/users/${userId}/chats`, currentCharacter.id || 'default_chat');
          await setDoc(chatDocRef, {
            messages: [...chatHistory, userMessage, { role: 'ai', text: aiText }],
            lastUpdated: serverTimestamp(),
            characterId: currentCharacter.id,
            characterName: currentCharacter.name,
          }, { merge: true });
        }

      } else {
        console.error('Estrutura de resposta inesperada da API Gemini:', result);
        setChatHistory((prev) => [...prev, { role: 'ai', text: 'Desculpe, não consegui gerar uma resposta. Tente novamente.' }]);
      }
    } catch (error) {
      console.error('Erro ao comunicar com a API Gemini:', error);
      setChatHistory((prev) => [...prev, { role: 'ai', text: 'Ocorreu um erro ao enviar a sua mensagem. Por favor, tente novamente.' }]);
    } finally {
      setIsLoading(false);
    }
  };

  // Handle character image file upload
  const handleCharacterImageUpload = (e) => {
    const file = e.target.files[0];
    if (file) {
      setCharacterImageFile(file);
      const reader = new FileReader();
      reader.onloadend = () => {
        setCharacterImagePreview(reader.result);
      };
      reader.readAsDataURL(file);
    } else {
      setCharacterImageFile(null);
      setCharacterImagePreview('');
    }
  };

  // Handle background image file upload
  const handleBackgroundImageUpload = (e) => {
    const file = e.target.files[0];
    if (file) {
      setBackgroundImageFile(file);
      const reader = new FileReader();
      reader.onloadend = () => {
        setBackgroundImagePreview(reader.result);
      };
      reader.readAsDataURL(file);
    } else {
      setBackgroundImageFile(null);
      setBackgroundImagePreview('');
    }
  };

  // Function to create a new character
  const handleCreateCharacter = async () => {
    if (!characterNameInput.trim() || !characterDescriptionInput.trim()) {
      alert('Por favor, preencha o nome e a descrição do personagem.');
      return;
    }

    const newCharacter = {
      id: Date.now().toString(), // Simple unique ID
      name: characterNameInput.trim(),
      description: characterDescriptionInput.trim(),
      // Use base64 string from preview if file is uploaded, otherwise default placeholder
      imageUrl: characterImagePreview || 'https://placehold.co/100x100/A78BFA/ffffff?text=AI',
      backgroundUrl: backgroundImagePreview || '', // Use base64 string from preview or empty
      createdAt: serverTimestamp(),
      createdBy: userId,
    };
    setCurrentCharacter(newCharacter);
    setCharacterNameInput('');
    setCharacterDescriptionInput('');
    setCharacterImageFile(null);
    setCharacterImagePreview('');
    setBackgroundImageFile(null);
    setBackgroundImagePreview('');
    setChatHistory([]); // Clear chat history for new character

    // Optional: Save character to Firestore
    if (db && userId) {
      try {
        const characterDocRef = doc(db, `artifacts/${appId}/users/${userId}/characters`, newCharacter.id);
        // WARNING: Storing large base64 images directly in Firestore documents is NOT recommended
        // due to the 1MB document limit. For production, consider Firebase Storage.
        await setDoc(characterDocRef, newCharacter);
        console.log('Personagem salvo no Firestore:', newCharacter.name);
      } catch (error) {
        console.error('Erro ao salvar personagem no Firestore:', error);
      }
    }
  };

  // Optional: Load characters and chat history from Firestore (example)
  useEffect(() => {
    if (db && userId && isAuthReady) {
      // Load characters
      const charactersCollectionRef = collection(db, `artifacts/${appId}/users/${userId}/characters`);
      const q = query(charactersCollectionRef, orderBy('createdAt', 'desc')); // Order by creation time

      const unsubscribeCharacters = onSnapshot(q, (snapshot) => {
        const loadedCharacters = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
        // You would typically display these characters for selection
        console.log("Personagens carregados:", loadedCharacters);
        // For this example, let's just set the first one as current if none is selected
        if (!currentCharacter && loadedCharacters.length > 0) {
          setCurrentCharacter(loadedCharacters[0]);
        }
      }, (error) => {
        console.error("Erro ao carregar personagens:", error);
      });

      // Load chat history for the current character if available
      if (currentCharacter && currentCharacter.id) {
        const chatDocRef = doc(db, `artifacts/${appId}/users/${userId}/chats`, currentCharacter.id);
        const unsubscribeChat = onSnapshot(chatDocRef, (docSnap) => {
          if (docSnap.exists()) {
            const chatData = docSnap.data();
            // Ensure messages are an array and filter out non-objects or malformed messages
            if (Array.isArray(chatData.messages)) {
              const validMessages = chatData.messages.filter(msg => msg && typeof msg.role === 'string' && typeof msg.text === 'string');
              setChatHistory(validMessages);
            }
          } else {
            setChatHistory([]); // No chat history for this character
          }
        }, (error) => {
          console.error("Erro ao carregar histórico de chat:", error);
        });
        return () => {
          unsubscribeCharacters();
          unsubscribeChat();
        }; // Cleanup listeners
      }
      return () => unsubscribeCharacters(); // Cleanup characters listener if no currentCharacter
    }
  }, [db, userId, isAuthReady, currentCharacter?.id]); // Re-run when db, userId, or currentCharacter.id changes


  return (
    <div
      className="min-h-screen bg-gradient-to-br from-purple-800 to-indigo-900 text-white font-inter flex flex-col items-center p-4 sm:p-6"
      style={{
        backgroundImage: currentCharacter && currentCharacter.backgroundUrl ? `url(${currentCharacter.backgroundUrl})` : 'none',
        backgroundSize: 'cover',
        backgroundPosition: 'center',
        backgroundAttachment: 'fixed',
      }}
    >
      <meta name="viewport" content="width=device-width, initial-scale=1.0" />
      <script src="https://cdn.tailwindcss.com"></script>
      <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;600;700&display=swap" rel="stylesheet" />

      <style>
        {`
          body { font-family: 'Inter', sans-serif; }
          .chat-message-container {
            display: flex;
            align-items: flex-start; /* Alinha o avatar com o topo da mensagem */
          }
          .chat-message.user {
            background-color: #4f46e5; /* Indigo-600 */
            align-self: flex-end;
            border-bottom-right-radius: 0;
          }
          .chat-message.ai {
            background-color: #6d28d9; /* Violet-700 */
            align-self: flex-start;
            border-bottom-left-radius: 0;
          }
          .chat-message {
            max-width: 80%;
            padding: 0.75rem 1rem;
            border-radius: 1.25rem;
            margin-bottom: 0.75rem;
            word-wrap: break-word;
          }
          .character-avatar {
            width: 40px;
            height: 40px;
            border-radius: 50%;
            object-fit: cover;
            margin-right: 10px;
            flex-shrink: 0; /* Evita que o avatar encolha */
          }
          .custom-scrollbar::-webkit-scrollbar {
            width: 8px;
          }
          .custom-scrollbar::-webkit-scrollbar-track {
            background: #555;
            border-radius: 10px;
          }
          .custom-scrollbar::-webkit-scrollbar-thumb {
            background: #888;
            border-radius: 10px;
          }
          .custom-scrollbar::-webkit-scrollbar-thumb:hover {
            background: #aaa;
          }
          .image-preview {
            width: 80px;
            height: 80px;
            object-fit: cover;
            border-radius: 10px;
            margin-top: 10px;
            margin-bottom: 10px;
            border: 2px solid #a78bfa; /* Purple-300 */
          }
        `}
      </style>

      <header className="w-full max-w-3xl text-center mb-8">
        <h1 className="text-4xl sm:text-5xl font-extrabold text-purple-200 drop-shadow-lg">
          Not PolyBuzz Copy
        </h1>
        <p className="text-sm text-purple-300 mt-2">
          Criado pelo Aqua Hoshino
        </p>
        {userId && (
          <p className="text-sm text-purple-300 mt-2">
            ID do Utilizador: <span className="font-mono text-purple-100 break-all">{userId}</span>
          </p>
        )}
      </header>

      <main className="w-full max-w-3xl bg-gray-800 bg-opacity-70 rounded-3xl shadow-2xl p-6 sm:p-8 flex flex-col flex-grow">
        {!currentCharacter ? (
          <div className="flex flex-col items-center justify-center flex-grow">
            <h2 className="text-3xl font-bold mb-6 text-purple-300">Crie o seu Personagem de IA</h2>
            <div className="w-full max-w-md">
              <input
                type="text"
                placeholder="Nome do Personagem (ex: Mago Sábio)"
                className="w-full p-3 mb-4 rounded-xl bg-gray-700 text-white placeholder-gray-400 focus:outline-none focus:ring-2 focus:ring-purple-500"
                value={characterNameInput}
                onChange={(e) => setCharacterNameInput(e.target.value)}
              />
              <textarea
                placeholder="Descrição do Personagem (ex: Um mago antigo com um senso de humor peculiar e conhecimento vasto sobre o universo.)"
                className="w-full p-3 mb-4 rounded-xl bg-gray-700 text-white placeholder-gray-400 h-32 resize-none focus:outline-none focus:ring-2 focus:ring-purple-500"
                value={characterDescriptionInput}
                onChange={(e) => setCharacterDescriptionInput(e.target.value)}
              ></textarea>
              {/* Input for character image file upload */}
              <label htmlFor="character-image-upload" className="block text-gray-300 text-sm font-bold mb-2">
                Foto do Personagem:
              </label>
              <input
                id="character-image-upload"
                type="file"
                accept="image/*"
                className="w-full p-3 mb-2 rounded-xl bg-gray-700 text-white placeholder-gray-400 focus:outline-none focus:ring-2 focus:ring-purple-500"
                onChange={handleCharacterImageUpload}
              />
              {characterImagePreview && (
                <img src={characterImagePreview} alt="Pré-visualização do Personagem" className="image-preview mx-auto" />
              )}
              {/* Input for background image file upload */}
              <label htmlFor="background-image-upload" className="block text-gray-300 text-sm font-bold mb-2 mt-4">
                Imagem de Fundo:
              </label>
              <input
                id="background-image-upload"
                type="file"
                accept="image/*"
                className="w-full p-3 mb-6 rounded-xl bg-gray-700 text-white placeholder-gray-400 focus:outline-none focus:ring-2 focus:ring-purple-500"
                onChange={handleBackgroundImageUpload}
              />
              {backgroundImagePreview && (
                <img src={backgroundImagePreview} alt="Pré-visualização do Fundo" className="image-preview mx-auto" />
              )}

              <button
                onClick={handleCreateCharacter}
                className="w-full bg-purple-600 hover:bg-purple-700 text-white font-bold py-3 px-6 rounded-xl transition duration-300 ease-in-out transform hover:scale-105 shadow-lg"
              >
                Criar Personagem
              </button>
            </div>
          </div>
        ) : (
          <div className="flex flex-col flex-grow h-full">
            <h2 className="text-2xl sm:text-3xl font-bold mb-4 text-purple-300 text-center">
              A conversar com {currentCharacter.name}
            </h2>
            <p className="text-center text-sm text-gray-400 mb-4 italic">
              "{currentCharacter.description}"
            </p>
            <div
              ref={chatContainerRef}
              className="flex-grow overflow-y-auto p-4 bg-gray-700 bg-opacity-50 rounded-2xl mb-4 custom-scrollbar"
              style={{ minHeight: '300px' }} // Ensure a minimum height for chat area
            >
              {chatHistory.length === 0 && (
                <p className="text-center text-gray-400 italic mt-8">
                  Diga olá ao seu novo personagem!
                </p>
              )}
              {chatHistory.map((msg, index) => (
                <div
                  key={index}
                  className={`chat-message-container flex ${msg.role === 'user' ? 'justify-end' : 'justify-start'} ${msg.role === 'ai' ? 'items-start' : 'items-end'}`}
                >
                  {/* Display character avatar for AI messages */}
                  {msg.role === 'ai' && (
                    <img
                      src={currentCharacter.imageUrl}
                      alt="Character Avatar"
                      className="character-avatar"
                      onError={(e) => { e.target.onerror = null; e.target.src = 'https://placehold.co/100x100/A78BFA/ffffff?text=AI'; }}
                    />
                  )}
                  <div className={`chat-message ${msg.role === 'user' ? 'user' : 'ai'}`}>
                    {msg.text}
                  </div>
                </div>
              ))}
              {isLoading && (
                <div className="flex justify-start chat-message-container">
                  {currentCharacter && (
                    <img
                      src={currentCharacter.imageUrl}
                      alt="Character Avatar"
                      className="character-avatar"
                      onError={(e) => { e.target.onerror = null; e.target.src = 'https://placehold.co/100x100/A78BFA/ffffff?text=AI'; }}
                    />
                  )}
                  <div className="chat-message ai animate-pulse">
                    <TypingIndicator /> {/* Use o componente TypingIndicator aqui */}
                  </div>
                </div>
              )}
            </div>

            <div className="flex items-center">
              <input
                type="text"
                placeholder="Escreva a sua mensagem..."
                className="flex-grow p-3 rounded-l-xl bg-gray-700 text-white placeholder-gray-400 focus:outline-none focus:ring-2 focus:ring-purple-500"
                value={userInput}
                onChange={(e) => setUserInput(e.target.value)}
                onKeyPress={(e) => e.key === 'Enter' && handleSendMessage()}
                disabled={isLoading}
              />
              <button
                onClick={handleSendMessage}
                className="bg-purple-600 hover:bg-purple-700 text-white font-bold py-3 px-6 rounded-r-xl transition duration-300 ease-in-out transform hover:scale-105 shadow-lg flex items-center justify-center"
                disabled={isLoading}
              >
                {isLoading ? (
                  <svg className="animate-spin h-5 w-5 text-white" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
                    <circle className="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" strokeWidth="4"></circle>
                    <path className="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
                  </svg>
                ) : (
                  'Enviar'
                )}
              </button>
            </div>
            <button
              onClick={() => setCurrentCharacter(null)}
              className="mt-4 w-full bg-gray-600 hover:bg-gray-700 text-white font-bold py-2 px-4 rounded-xl transition duration-300 ease-in-out shadow-md"
            >
              Criar Novo Personagem
            </button>
          </div>
        )}
      </main>
    </div>
  );
}

export default App;
