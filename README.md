// Importa hooks do React e funções do Firebase para autenticação e Firestore
import { createContext, useContext, useEffect, useState } from "react";
import {
  onAuthStateChanged, // Observa mudanças no estado de autenticação
  createUserWithEmailAndPassword, // Cria um usuário com email e senha
  signInWithEmailAndPassword, // Faz login com email e senha
  signOut, // Faz logout
} from 'firebase/auth';
import { auth, db } from "../firebaseConfig"; // Importa as instâncias configuradas do Firebase
import { doc, getDoc, setDoc } from 'firebase/firestore'; // Firestore: leitura e escrita de documentos

// Cria o contexto de autenticação que será usado em toda a aplicação
export const AuthContext = createContext();

// Provedor do contexto de autenticação, que envolve a aplicação e fornece os métodos e dados de auth
export const AuthContextProvider = ({ children }) => {
  const [user, setUser] = useState(null); // Armazena os dados do usuário autenticado
  const [isAuthenticated, setIsAuthenticated] = useState(undefined); // Estado para verificar se o usuário está autenticado

  useEffect(() => {
    // Listener do Firebase que detecta mudanças na autenticação
    const unsub = onAuthStateChanged(auth, (user) => {
      if (user) {
        setIsAuthenticated(true);
        setUser(user);
        updateUserData(user.uid); // Atualiza os dados do Firestore com mais informações do usuário
      } else {
        setIsAuthenticated(false);
        setUser(null);
      }
    });

    return unsub; // Remove o listener quando o componente é desmontado
  }, []);

  // Busca dados adicionais do usuário no Firestore (como username e profileUrl)
  const updateUserData = async (userId) => {
    const docRef = doc(db, 'users', userId); // Referência ao documento do usuário
    const docSnap = await getDoc(docRef); // Tenta buscar o documento

    if (docSnap.exists()) {
      let data = docSnap.data();
      // Atualiza o estado do usuário com os dados do Firestore
      setUser({
        ...user,
        username: data.username,
        profileUrl: data.profileUrl,
        userId: data.userId,
      });
    }
  };

  // Função de login usando Firebase Auth
  const login = async (email, password) => {
    try {
      const response = await signInWithEmailAndPassword(auth, email, password);
      return { success: true };
    } catch (e) {
      // Trata erros comuns de login com mensagens personalizadas
      let msg = e.message;
      if (msg.includes('(auth/invalid-email)')) msg = 'E-mail inválido';
      if (msg.includes('(auth/invalid-credential)')) msg = 'E-mail ou Senha errada';
      return { success: false, msg };
    }
  };

  // Função de logout usando Firebase Auth
  const logout = async () => {
    try {
      await signOut(auth);
      return { success: true };
    } catch (e) {
      return { success: false, msg: e.message, error: e };
    }
  };

  // Função de registro (signup) de novo usuário
  const register = async (email, password, username, profileUrl) => {
    try {
      const response = await createUserWithEmailAndPassword(auth, email, password);
      console.log('response.user :', response?.user);

      // Armazena dados adicionais do usuário no Firestore
      await setDoc(doc(db, "users", response?.user?.uid), {
        username,
        profileUrl,
        userId: response?.user?.uid,
      });

      return { success: true, data: response?.user };
    } catch (e) {
      // Trata erros comuns de registro
      let msg = e.message;
      if (msg.includes('(auth/invalid-email)')) msg = 'E-mail inválido';
      if (msg.includes('(auth/email-already-in-use)')) msg = 'Esse e-mail já está em uso';
      return { success: false, msg };
    }
  };

  // O provider disponibiliza os métodos e estados via contexto
  return (
    <AuthContext.Provider value={{ user, isAuthenticated, login, register, logout }}>
      {children}
    </AuthContext.Provider>
  );
};

// Hook customizado para acessar facilmente o contexto de autenticação
export const useAuth = () => {
  const value = useContext(AuthContext);

  // Garante que o hook seja usado dentro do provider
  if (!value) {
    throw new Error('useAuth must be wrapped inside AuthContextProvider');
  }

  return value;
};

# Comentarios-sobre-codigo
