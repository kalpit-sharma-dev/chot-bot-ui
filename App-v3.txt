import React, { useState, useRef, useEffect } from 'react';
import {
  SafeAreaView,
  TextInput,
  FlatList,
  Text,
  View,
  KeyboardAvoidingView,
  Platform,
  ActivityIndicator,
  StyleSheet,
  TouchableOpacity,
  Alert,
} from 'react-native';
import { Ionicons } from '@expo/vector-icons';
import Constants from 'expo-constants';
import AsyncStorage from '@react-native-async-storage/async-storage';

type Message = {
  id: string;
  role: 'user' | 'assistant';
  content: string;
  isStreaming?: boolean;
};

type AuthResponse = {
  token: string;
  session_id: string;
  expires_at: number;
};

const API_BASE_URL = 'http://192.168.0.186:8080';
const TOKEN_KEY = 'auth_token';
const SESSION_KEY = 'session_id';
const TOKEN_EXPIRY_KEY = 'token_expiry';

export default function App() {
  const [messages, setMessages] = useState<Message[]>([
    {
      id: '1',
      role: 'assistant',
      content: 'Hi! How can I help you with your banking today?',
    },
  ]);
  const [input, setInput] = useState('');
  const [loading, setLoading] = useState(false);
  const [authLoading, setAuthLoading] = useState(true);
  const [isAuthenticated, setIsAuthenticated] = useState(false);
  
  const flatListRef = useRef<FlatList>(null);
  const streamingRef = useRef(false);
  const currentStreamingIdRef = useRef<string | null>(null);
  const abortControllerRef = useRef<AbortController | null>(null);
  
  // Authentication state
  const tokenRef = useRef<string | null>(null);
  const sessionIdRef = useRef<string | null>(null);
  const tokenExpiryRef = useRef<number | null>(null);

  useEffect(() => {
    initializeAuth();
  }, []);

  useEffect(() => {
    if (messages.length > 1) {
      flatListRef.current?.scrollToEnd({ animated: true });
    }
  }, [messages]);

  // Cleanup on unmount
  useEffect(() => {
    return () => {
      if (abortControllerRef.current) {
        abortControllerRef.current.abort();
      }
    };
  }, []);

  const initializeAuth = async () => {
    try {
      setAuthLoading(true);
      
      // Try to load existing token
      const [storedToken, storedSessionId, storedExpiry] = await Promise.all([
        AsyncStorage.getItem(TOKEN_KEY),
        AsyncStorage.getItem(SESSION_KEY),
        AsyncStorage.getItem(TOKEN_EXPIRY_KEY),
      ]);

      const now = Date.now() / 1000;
      const expiry = storedExpiry ? parseInt(storedExpiry) : 0;

      // Check if token exists and is not expired
      if (storedToken && storedSessionId && expiry > now) {
        console.log('Using existing valid token');
        tokenRef.current = storedToken;
        sessionIdRef.current = storedSessionId;
        tokenExpiryRef.current = expiry;
        setIsAuthenticated(true);
      } else {
        console.log('Getting new authentication token');
        await authenticate();
      }
    } catch (error) {
      console.error('Auth initialization error:', error);
      await authenticate(); // Fallback to new auth
    } finally {
      setAuthLoading(false);
    }
  };

  const authenticate = async () => {
    try {
      console.log('Authenticating with server...');
      const response = await fetch(`${API_BASE_URL}/auth`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
        },
      });

      if (!response.ok) {
        throw new Error(`Authentication failed: ${response.status}`);
      }

      const authData: AuthResponse = await response.json();
      
      // Store authentication data
      tokenRef.current = authData.token;
      sessionIdRef.current = authData.session_id;
      tokenExpiryRef.current = authData.expires_at;

      // Persist to storage
      await Promise.all([
        AsyncStorage.setItem(TOKEN_KEY, authData.token),
        AsyncStorage.setItem(SESSION_KEY, authData.session_id),
        AsyncStorage.setItem(TOKEN_EXPIRY_KEY, authData.expires_at.toString()),
      ]);

      setIsAuthenticated(true);
      console.log('Authentication successful');
      
    } catch (error) {
      console.error('Authentication error:', error);
      Alert.alert(
        'Authentication Failed',
        'Failed to connect to the server. Please check your connection and try again.',
        [{ text: 'Retry', onPress: authenticate }]
      );
    }
  };

  const checkTokenExpiry = async () => {
    const now = Date.now() / 1000;
    if (tokenExpiryRef.current && tokenExpiryRef.current <= now + 300) { // Refresh 5 minutes before expiry
      console.log('Token expiring soon, refreshing...');
      await authenticate();
    }
  };

  const sendMessage = async () => {
    if (!input.trim() || loading || !isAuthenticated) return;

    // Check token expiry before sending
    await checkTokenExpiry();

    // Cancel any ongoing stream
    if (abortControllerRef.current) {
      abortControllerRef.current.abort();
    }

    const userMessage: Message = {
      id: Date.now().toString(),
      role: 'user',
      content: input,
    };

    const currentInput = input;
    setMessages((prev) => [...prev, userMessage]);
    setInput('');
    setLoading(true);

    const assistantMessageId = Date.now().toString() + '-ai';
    currentStreamingIdRef.current = assistantMessageId;
    
    // Add empty assistant message for streaming
    setMessages((prev) => [
      ...prev,
      {
        id: assistantMessageId,
        role: 'assistant',
        content: '',
        isStreaming: true,
      },
    ]);
    streamingRef.current = true;

    // Create new AbortController for this request
    abortControllerRef.current = new AbortController();

    try {
      console.log('Starting authenticated streaming...');
      await streamWithXHR(currentInput, assistantMessageId);
    } catch (err) {
      console.error("Streaming error:", err);
      
      // Check if it's an auth error
      if (err instanceof Error && (err.message.includes('401') || err.message.includes('Unauthorized'))) {
        console.log('Authentication expired, refreshing...');
        try {
          await authenticate();
          // Retry the message after re-authentication
          if (streamingRef.current && currentStreamingIdRef.current === assistantMessageId) {
            await streamWithXHR(currentInput, assistantMessageId);
            return;
          }
        } catch (authError) {
          console.error('Re-authentication failed:', authError);
        }
      }
      
      // Don't show error if request was aborted intentionally
      if (err instanceof Error && err.name === 'AbortError') {
        console.log('Request was aborted');
        return;
      }
      
      setMessages(prev => prev.map(msg => 
        msg.id === assistantMessageId 
          ? { 
              ...msg, 
              content: `Sorry, I encountered an error: ${err instanceof Error ? err.message : 'Unknown error'}. Please check if the server is running and accessible.`,
              isStreaming: false 
            } 
          : msg
      ));
    } finally {
      setLoading(false);
      streamingRef.current = false;
      currentStreamingIdRef.current = null;
      abortControllerRef.current = null;
    }
  };

  const streamWithXHR = async (message: string, messageId: string) => {
    return new Promise<void>((resolve, reject) => {
      if (!tokenRef.current) {
        reject(new Error('No authentication token available'));
        return;
      }

      const xhr = new XMLHttpRequest();
      let lastProcessedIndex = 0;
      let buffer = '';

      xhr.open('POST', `${API_BASE_URL}/chat`, true);
      xhr.setRequestHeader('Content-Type', 'application/json');
      xhr.setRequestHeader('Accept', 'text/event-stream');
      xhr.setRequestHeader('Cache-Control', 'no-cache');

      // Set response type for better streaming performance
      xhr.responseType = 'text';

      const requestBody = JSON.stringify({ 
        message,
        token: tokenRef.current,
        session_id: sessionIdRef.current
      });

      xhr.onreadystatechange = () => {
        if (xhr.readyState === XMLHttpRequest.HEADERS_RECEIVED) {
          console.log('Headers received, status:', xhr.status);
          if (xhr.status === 401) {
            reject(new Error('Unauthorized - token may be expired'));
            return;
          } else if (xhr.status !== 200) {
            reject(new Error(`HTTP ${xhr.status}: ${xhr.statusText}`));
            return;
          }
        }
      };

      xhr.onprogress = () => {
        if (!streamingRef.current || currentStreamingIdRef.current !== messageId) {
          xhr.abort();
          return;
        }

        // Get new data from the response
        const fullResponse = xhr.responseText;
        const newData = fullResponse.substring(lastProcessedIndex);
        lastProcessedIndex = fullResponse.length;

        if (!newData) return;

        // Add new data to buffer
        buffer += newData;

        // Process complete lines in buffer
        const lines = buffer.split('\n');
        
        // Keep the last incomplete line in buffer
        buffer = lines.pop() || '';

        // Process complete lines
        for (const line of lines) {
          if (line.startsWith('data: ')) {
            const data = line.substring(6);
            
            if (data.trim() === '[DONE]') {
              console.log('Stream completed');
              setMessages(prev => prev.map(msg => 
                msg.id === messageId 
                  ? { ...msg, isStreaming: false } 
                  : msg
              ));
              resolve();
              return;
            }

            if (data.trim()) {
              // Check for error messages
              if (data.startsWith('Error:') || data === 'Request timeout') {
                setMessages(prev => prev.map(msg => 
                  msg.id === messageId 
                    ? { ...msg, content: data, isStreaming: false } 
                    : msg
                ));
                reject(new Error(data));
                return;
              }

              // Add token to message content
              setMessages(prev => prev.map(msg => 
                msg.id === messageId 
                  ? { ...msg, content: msg.content + data } 
                  : msg
              ));
            }
          }
        }
      };

      xhr.onload = () => {
        console.log('XHR stream completed with status:', xhr.status);
        if (xhr.status === 200) {
          setMessages(prev => prev.map(msg => 
            msg.id === messageId 
              ? { ...msg, isStreaming: false } 
              : msg
          ));
          resolve();
        } else if (xhr.status === 401) {
          reject(new Error('Unauthorized - token may be expired'));
        } else {
          reject(new Error(`HTTP ${xhr.status}: ${xhr.statusText}`));
        }
      };

      xhr.onerror = () => {
        console.error('XHR error occurred');
        reject(new Error('Network request failed'));
      };

      xhr.ontimeout = () => {
        console.error('XHR timeout');
        reject(new Error('Request timeout'));
      };

      // Set timeout for the request
      xhr.timeout = 300000; // 5 minutes

      console.log('Sending authenticated request...');
      xhr.send(requestBody);

      // Store XHR reference for cancellation
      abortControllerRef.current = {
        abort: () => {
          console.log('Aborting XHR request');
          xhr.abort();
        },
        signal: null as any
      } as AbortController;
    });
  };

  const stopStreaming = () => {
    if (abortControllerRef.current) {
      abortControllerRef.current.abort();
    }
    streamingRef.current = false;
    currentStreamingIdRef.current = null;
    setLoading(false);

    // Mark current streaming message as complete
    setMessages(prev => prev.map(msg => 
      msg.isStreaming ? { ...msg, isStreaming: false } : msg
    ));
  };

  const clearSession = async () => {
    try {
      await Promise.all([
        AsyncStorage.removeItem(TOKEN_KEY),
        AsyncStorage.removeItem(SESSION_KEY),
        AsyncStorage.removeItem(TOKEN_EXPIRY_KEY),
      ]);
      
      tokenRef.current = null;
      sessionIdRef.current = null;
      tokenExpiryRef.current = null;
      setIsAuthenticated(false);
      setMessages([{
        id: '1',
        role: 'assistant',
        content: 'Hi! How can I help you with your banking today?',
      }]);
      
      await authenticate();
    } catch (error) {
      console.error('Error clearing session:', error);
    }
  };

  const renderItem = ({ item }: { item: Message }) => (
    <View
      style={[
        styles.messageContainer,
        item.role === 'user' ? styles.userMessage : styles.aiMessage,
      ]}
    >
      <Text style={[
        styles.messageText,
        item.role === 'user' && styles.userMessageText
      ]}>
        {item.content}
        {item.isStreaming && (
          <Text style={styles.typingIndicator}>
            <Text style={styles.cursor}>|</Text>
          </Text>
        )}
      </Text>
    </View>
  );

  if (authLoading) {
    return (
      <SafeAreaView style={styles.container}>
        <View style={styles.loadingContainer}>
          <ActivityIndicator size="large" color="#007AFF" />
          <Text style={styles.loadingText}>Connecting to server...</Text>
        </View>
      </SafeAreaView>
    );
  }

  if (!isAuthenticated) {
    return (
      <SafeAreaView style={styles.container}>
        <View style={styles.loadingContainer}>
          <Text style={styles.errorText}>Authentication Required</Text>
          <TouchableOpacity onPress={authenticate} style={styles.retryButton}>
            <Text style={styles.retryButtonText}>Retry Connection</Text>
          </TouchableOpacity>
        </View>
      </SafeAreaView>
    );
  }

  return (
    <SafeAreaView style={styles.container}>
      <View style={styles.header}>
        <Text style={styles.heading}>AI Banking Assistant</Text>
        <View style={styles.headerButtons}>
          {loading && (
            <TouchableOpacity onPress={stopStreaming} style={styles.stopButton}>
              <Text style={styles.stopButtonText}>Stop</Text>
            </TouchableOpacity>
          )}
          <TouchableOpacity onPress={clearSession} style={styles.refreshButton}>
            <Ionicons name="refresh" size={20} color="white" />
          </TouchableOpacity>
        </View>
      </View>
      
      <KeyboardAvoidingView
        style={styles.keyboardAvoidingView}
        behavior={Platform.OS === 'ios' ? 'padding' : 'height'}
        keyboardVerticalOffset={Platform.OS === 'ios' ? 60 : 0}
      >
        <View style={styles.content}>
          <FlatList
            ref={flatListRef}
            data={messages}
            renderItem={renderItem}
            keyExtractor={(item) => item.id}
            contentContainerStyle={styles.messageList}
            keyboardDismissMode="interactive"
            keyboardShouldPersistTaps="handled"
            automaticallyAdjustContentInsets={false}
            onContentSizeChange={() => 
              setTimeout(() => flatListRef.current?.scrollToEnd({ animated: true }), 50)
            }
            onLayout={() => 
              setTimeout(() => flatListRef.current?.scrollToEnd({ animated: true }), 50)
            }
          />
          
          <View style={styles.inputContainer}>
            <TextInput
              value={input}
              onChangeText={setInput}
              placeholder="Ask about your banking needs..."
              style={styles.textInput}
              editable={!loading}
              returnKeyType="send"
              onSubmitEditing={sendMessage}
              multiline={false}
            />
            <TouchableOpacity 
              onPress={loading ? stopStreaming : sendMessage} 
              style={[
                styles.sendButton,
                !input.trim() && !loading && styles.sendButtonDisabled
              ]}
              disabled={!input.trim() && !loading}
            >
              {loading ? (
                <Ionicons
                  name="stop"
                  size={24}
                  color="#FF3B30"
                />
              ) : (
                <Ionicons
                  name="send"
                  size={24}
                  color={!input.trim() ? '#999' : '#007AFF'}
                />
              )}
            </TouchableOpacity>
          </View>
        </View>
      </KeyboardAvoidingView>
    </SafeAreaView>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: '#f8f9fa',
  },
  loadingContainer: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
    padding: 20,
  },
  loadingText: {
    marginTop: 16,
    fontSize: 16,
    color: '#666',
    textAlign: 'center',
  },
  errorText: {
    fontSize: 18,
    color: '#FF3B30',
    textAlign: 'center',
    marginBottom: 20,
  },
  retryButton: {
    backgroundColor: '#007AFF',
    paddingHorizontal: 20,
    paddingVertical: 12,
    borderRadius: 8,
  },
  retryButtonText: {
    color: 'white',
    fontSize: 16,
    fontWeight: '600',
  },
  header: {
    paddingTop: Constants.statusBarHeight,
    backgroundColor: '#007AFF',
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.1,
    shadowRadius: 4,
    elevation: 3,
    flexDirection: 'row',
    alignItems: 'center',
    justifyContent: 'space-between',
    paddingHorizontal: 15,
  },
  heading: {
    fontSize: 20,
    fontWeight: '600',
    paddingVertical: 15,
    color: 'white',
    flex: 1,
  },
  headerButtons: {
    flexDirection: 'row',
    alignItems: 'center',
    gap: 10,
  },
  stopButton: {
    backgroundColor: 'rgba(255, 255, 255, 0.2)',
    paddingHorizontal: 12,
    paddingVertical: 6,
    borderRadius: 15,
  },
  stopButtonText: {
    color: 'white',
    fontSize: 14,
    fontWeight: '600',
  },
  refreshButton: {
    padding: 8,
    backgroundColor: 'rgba(255, 255, 255, 0.2)',
    borderRadius: 20,
  },
  keyboardAvoidingView: {
    flex: 1,
  },
  content: {
    flex: 1,
    justifyContent: 'space-between',
  },
  messageList: {
    flexGrow: 1,
    paddingHorizontal: 15,
    paddingVertical: 10,
  },
  messageContainer: {
    borderRadius: 18,
    marginVertical: 3,
    padding: 12,
    maxWidth: '85%',
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 1 },
    shadowOpacity: 0.05,
    shadowRadius: 2,
    elevation: 1,
  },
  userMessage: {
    alignSelf: 'flex-end',
    backgroundColor: '#007AFF',
  },
  aiMessage: {
    alignSelf: 'flex-start',
    backgroundColor: 'white',
    borderWidth: 1,
    borderColor: '#e1e5e9',
  },
  messageText: {
    fontSize: 16,
    lineHeight: 22,
    color: '#333',
  },
  userMessageText: {
    color: 'white',
  },
  typingIndicator: {
    marginLeft: 2,
  },
  cursor: {
    color: '#007AFF',
    fontWeight: 'bold',
    fontSize: 18,
  },
  inputContainer: {
    flexDirection: 'row',
    alignItems: 'flex-end',
    paddingHorizontal: 15,
    paddingVertical: 10,
    borderTopColor: '#e1e5e9',
    borderTopWidth: 1,
    backgroundColor: 'white',
    paddingBottom: Platform.OS === 'ios' ? 10 : 15,
  },
  textInput: {
    flex: 1,
    borderColor: '#e1e5e9',
    borderWidth: 1,
    borderRadius: 22,
    paddingHorizontal: 16,
    paddingVertical: 12,
    fontSize: 16,
    backgroundColor: '#f8f9fa',
    marginRight: 10,
    maxHeight: 100,
    minHeight: 44,
  },
  sendButton: {
    padding: 12,
    borderRadius: 22,
    backgroundColor: '#f0f0f0',
  },
  sendButtonDisabled: {
    opacity: 0.5,
  },
});
