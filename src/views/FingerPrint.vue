<!-- <template>
    <div>
        <h1>Fingerprint</h1>
        <p>This page is under construction.</p>
        <p>It will be used to manage fingerprint data.</p>
        <p>Stay tuned for updates!</p>

        <div class="flex flex-col items-center gap-4">
    <n-button type="primary" @click="iniciarSesionBiometrico">
      Iniciar sesión con huella
    </n-button>
  </div>
    </div>
</template>

<script setup lang="ts">
import { startAuthentication } from '@simplewebauthn/browser';
import axios from 'axios'
import { useNotification, NButton } from 'naive-ui'

const notification = useNotification();
const BASE_URL = 'http://localhost:3006/auth-huella'

const userId = 'usuario1'; // En prod, usa el ID real del usuario

async function iniciarSesionBiometrico() {
  try {
    // Paso 1: obtener opciones de autenticación desde el backend
    const { data: options } = await axios.get(`${BASE_URL}/authentication-options`, {
      params: { userId },
    });

    // Paso 2: iniciar autenticación usando WebAuthn API
    const credential = await startAuthentication(options);

    // Paso 3: enviar la credencial al backend para verificación
    const { data: result } = await axios.post(`${BASE_URL}/verify-authentication`, {
      userId,
      credential,
    });

    if (result.success) {
      notification.success({
        title: 'Sesión iniciada',
        content: 'Autenticación biométrica exitosa',
      });
      // redirigir, emitir evento, etc.
    } else {
      notification.error({
        title: 'Fallo de autenticación',
        content: 'No se pudo verificar la huella',
      });
    }
  } catch (err: any) {
    console.error(err);
    notification.error({
      title: 'Error',
      content: err?.message || 'No se pudo iniciar sesión biométrica',
    });
  }
}

</script>

<style scoped>

</style> -->

<script setup lang="ts">
import { ref } from 'vue';
import { client } from '@passwordless-id/webauthn';
import axios from 'axios';

const username = ref('');
const message = ref('');
const loggedInUser = ref(null);
const api = axios.create({
  baseURL: 'https://sop25.local:3007', // URL de tu backend NestJS
  // baseURL: 'http://localhost:3006', // URL de tu backend NestJS
});

const handleRegister = async () => {
  message.value = '';
  if (!username.value) {
    message.value = 'Por favor, introduce un nombre de usuario.';
    return;
  }

  try {
    // 1. Obtener el challenge del servidor
    const { data: { challenge } } = await api.get(`/passwordless-id/register/challenge?username=${username.value}`);
    console.log('Challenge recibido:', challenge);
    // 2. Usar la librería cliente para registrar la huella
    const registration = await client.register({
      challenge: challenge,
      userVerification: 'required',
      user: {
        id: username.value,
        name: username.value,
        displayName: username.value
      },
      // authenticatorType: 'platform',
      discoverable: 'required'
    });

    // 3. Enviar el resultado de vuelta al servidor para verificación
    const { data: verification } = await api.post('/passwordless-id/register/verify', {
      username: username.value,
      registration,
    });

    if (verification.success) {
      message.value = `✅ ¡Registro exitoso para ${verification.user.username}! Ahora puedes iniciar sesión.`;
    } else {
        message.value = '❌ Falló la verificación del registro.';
    }
  } catch (error:any) {
    console.error('Error durante el registro:', error);
    message.value = `❌ Error: ${error.response?.data?.message || error.message}`;
  }
};

const handleLogin = async () => {
  message.value = '';
  if (!username.value) {
    message.value = 'Por favor, introduce tu nombre de usuario.';
    return;
  }

  try {
    // 1. Obtener el challenge del servidor
    const { data: { challenge } } = await api.get(`/passwordless-id/login/challenge?username=${username.value}`);

    // 2. Usar la librería cliente para autenticar con la huella
    const authentication = await client.authenticate({
      challenge: challenge,
      userVerification: 'required',
    });

    // 3. Enviar el resultado al servidor para verificación
    const { data: verification } = await api.post('/passwordless-id/login/verify', {
      username: username.value,
      authentication,
    });

    if (verification.success) {
      loggedInUser.value = verification.user.username;
      message.value = '🎉 ¡Inicio de sesión exitoso!';
    } else {
      message.value = '❌ Falló la verificación del inicio de sesión.';
    }
  } catch (error:any) {
    message.value = `❌ Error: ${error.response?.data?.message || error.message}`;
  }
};

const handleLogout = () => {
  loggedInUser.value = null;
  username.value = '';
  message.value = '';
}
</script>

<template>
  <div class="container">
    <div v-if="!loggedInUser">
      <h1>Autenticación con Huella Dactilar (WebAuthn)</h1>
      <p>
        Introduce un nombre de usuario para registrarte o iniciar sesión con tu huella dactilar.
      </p>
      <input
        v-model="username"
        type="text"
        placeholder="Nombre de usuario"
        :disabled="!!loggedInUser"
      />
      <div class="buttons">
        <button @click="handleRegister">Registrar Huella</button>
        <button @click="handleLogin">Iniciar Sesión con Huella</button>
      </div>
    </div>
    <div v-else>
      <h1>Bienvenido, {{ loggedInUser }}!</h1>
      <button @click="handleLogout">Cerrar Sesión</button>
    </div>

    <p v-if="message" class="message">{{ message }}</p>
  </div>
</template>

<style>
/* ... (agrega tus estilos aquí) ... */
.container {
  max-width: 500px;
  margin: 50px auto;
  padding: 20px;
  border: 1px solid #ccc;
  border-radius: 8px;
  text-align: center;
}
input {
  display: block;
  width: 80%;
  padding: 10px;
  margin: 15px auto;
}
.buttons {
  display: flex;
  justify-content: center;
  gap: 10px;
}
button {
  padding: 10px 20px;
  cursor: pointer;
}
.message {
    margin-top: 20px;
    font-weight: bold;
}
</style>