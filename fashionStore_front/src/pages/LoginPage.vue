<template>
  <div class="login-bg">
    <div class="login-card">
      <div @click="goToHome" class="logo-container q-mb-md">
        <q-img src="/img/Logo.png" style="width: 120px; height: 120px; object-fit: contain;" />
        <q-tooltip>Volver a la tienda</q-tooltip>
      </div>

      <q-form @submit.prevent="onLogin">
        <q-input
          v-model="username"
          label="Username"
          type="text"
          dense
          outlined
          class="q-mb-md"
          prepend-inner-icon="person"
        />

        <q-input
          v-model="password"
          label="Password"
          type="password"
          dense
          outlined
          class="q-mb-md"
          prepend-inner-icon="lock"
        />

        <div class="row items-center q-mb-md">
          <q-checkbox v-model="remember" label="Recuérdame" />
          <q-space />
        </div>

        <q-btn
          type="submit"
          label="LOGIN"
          color="purple-7"
          class="full-width login-btn q-mb-md"
        />
      </q-form>

      <div class="text-center q-mt-md">
        <span class="text-black-4">¿No tienes cuenta?</span>
        <q-btn flat label="Regístrate" color="purple-4" @click="onRegister" />

        <q-btn
          flat
          color="purple-4"
          label="Olvidaste la contraseña?"
          @click="abrirDialogoRecuperarContrasenha"
        />
      </div>
    </div>
  </div>

  <!-- Diálogo para recuperar contraseña -->
  <q-dialog v-model="showForgotDialog">
    <q-card style="min-width: 350px">
      <q-card-section>
        <div class="text-h6">Recuperar contraseña</div>
      </q-card-section>

      <q-card-section v-if="!correoValido">
        <q-input
          v-model="forgotEmail"
          label="Correo electrónico"
          type="email"
          outlined
          dense
          autofocus
        />
      </q-card-section>

      <q-card-section v-if="correoValido">
        <q-input
          v-model="nuevaContrasenna"
          label="Nueva contraseña"
          type="password"
          outlined
          dense
          class="q-mb-md"
        />
        <q-input
          v-model="confirmarContrasenna"
          label="Confirmar contraseña"
          type="password"
          outlined
          dense
        />
      </q-card-section>

      <q-card-actions align="right">
        <q-btn flat label="Cancelar" v-close-popup />
        <q-btn
          flat
          label="Enviar"
          color="primary"
          @click="correoValido ? cambiarContrasenna() : verificarCorreo()"
        />
      </q-card-actions>
    </q-card>
  </q-dialog>

  <DialogLoad :dialogLoad="dialogLoad" />
</template>

<script setup>
import { ref, onMounted } from 'vue'
import { useRouter } from 'vue-router'
import { useQuasar } from 'quasar'

import { loadGet, saveDataPronosticoEnviarObjeto } from 'src/assets/js/util/funciones'
import { Error } from 'src/assets/js/util/notify'
import DialogLoad from 'src/components/DialogBoxes/DialogLoad.vue'
import { api } from 'src/boot/axios'

// refs reactivas
const username = ref('')
const password = ref('')
const remember = ref(false)
const dialogLoad = ref(false)

const forgotEmail = ref('')
const nuevaContrasenna = ref('')
const confirmarContrasenna = ref('')
const correoValido = ref(false)

const showForgotDialog = ref(false)

const router = useRouter()
const $q = useQuasar()

// ✅ Función para verificar si el token es válido y redirigir si es necesario
// Esta función se usa solo cuando el usuario navega directamente a /login
const verificarTokenExistente = () => {
  // Buscar token en localStorage o sessionStorage
  const token = localStorage.getItem('token') || sessionStorage.getItem('token')
  const exp = localStorage.getItem('token_exp') || sessionStorage.getItem('token_exp')

  if (token && exp) {
    const expirationDate = new Date(exp)
    const ahora = new Date()

    if (expirationDate > ahora) {
      // Token válido, redirigir al perfil usando replace para evitar historial
      // Importante: usar nextTick para asegurar que el componente esté montado
      setTimeout(() => {
        router.replace({ name: 'Perfil' })
      }, 0)
      return true
    } else {
      // Token expirado, limpiar ambos almacenamientos
      localStorage.removeItem('token')
      localStorage.removeItem('token_exp')
      sessionStorage.removeItem('token')
      sessionStorage.removeItem('token_exp')
      $q.notify({
        type: 'warning',
        message: 'Tu sesión ha expirado. Por favor, inicia sesión nuevamente.'
      })
    }
  }
  return false
}

// ✅ Verificar sesión al cargar el componente
// NOTA: El router guard ya maneja la redirección de usuarios autenticados
// por lo que NO necesitamos verificar el token aquí para evitar bucles
// onMounted(() => {
//   verificarTokenExistente()
// })

// Métodos
const onLogin = async () => {
  await login()
}

const onRegister = () => {
  router.push('/register')
}

// Función de login real
const login = async () => {
  const url = 'Autenticacion/Login'
  const payload = {
    username: username.value,
    contrasenna: password.value
  }

  await saveDataPronosticoEnviarObjeto(url, payload, dialogLoad).then(
    async (respuesta) => {
      if (!!respuesta?.mensajeError) {
        Error(respuesta?.mensajeError)
      } else {
        let token = respuesta?.resultado?.result?.token
        let exp = respuesta?.resultado?.result?.fechaExpiracion

        if (token) {
          // ✅ Guardar según el estado de "Recuérdame"
          if (remember.value) {
            // Guardar en localStorage (persistente entre sesiones del navegador)
            localStorage.setItem('token', token)
            localStorage.setItem('token_exp', exp)
          } else {
            // Guardar en sessionStorage (solo para la pestaña actual)
            sessionStorage.setItem('token', token)
            sessionStorage.setItem('token_exp', exp)
          }
        }

        $q.notify({ type: 'positive', message: 'Login exitoso' })
        router.push({ name: 'Perfil' })
      }
    }
  )
}

const loginDesdeRecuperacion = async (usuario, contrasenha) => {
  const url = 'Autenticacion/Login'
  const payload = {
    username: usuario,
    contrasenna: contrasenha
  }

  await saveDataPronosticoEnviarObjeto(url, payload, dialogLoad).then(
    async (respuesta) => {
      if (!!respuesta?.mensajeError) {
        Error(respuesta?.mensajeError)
      } else {
        let token = respuesta?.resultado?.result?.token
        let exp = respuesta?.resultado?.result?.fechaExpiracion

        if (token) {
          // ✅ Aplicar la misma preferencia de "Recuérdame"
          if (remember.value) {
            localStorage.setItem('token', token)
            localStorage.setItem('token_exp', exp)
          } else {
            sessionStorage.setItem('token', token)
            sessionStorage.setItem('token_exp', exp)
          }
        }

        $q.notify({ type: 'positive', message: 'Login exitoso' })
        router.push({ name: 'Perfil' })
      }
    }
  )
}

function goToHome() {
  router.push('/')
}

const verificarCorreo = async () => {
  if (!forgotEmail.value) {
    $q.notify({ type: 'negative', message: 'Introduce tu correo' })
    return
  }
  const url = 'Autenticacion/VerificarCorreo'
  const payload = { correo: forgotEmail.value }
  try {
    const response = await saveDataPronosticoEnviarObjeto(url, payload, dialogLoad)
    if (response.resultado.existe) {
      correoValido.value = true
      $q.notify({ type: 'positive', message: 'Correo válido, ingresa tu nueva contraseña' })
    }
  } catch (err) {
    $q.notify({ type: 'negative', message: 'Correo no encontrado' })
  }
}

const cambiarContrasenna = async () => {
  if (!nuevaContrasenna.value || !confirmarContrasenna.value) {
    $q.notify({ type: 'negative', message: 'Completa ambos campos' })
    return
  }
  if (nuevaContrasenna.value !== confirmarContrasenna.value) {
    $q.notify({ type: 'negative', message: 'Las contraseñas no coinciden' })
    return
  }
  const url = 'Usuario/CambiarContrasennaDesdeRecuperar'
  const payload = {
    correo: forgotEmail.value,
    nuevaContrasenna: nuevaContrasenna.value,
    contrasennaConfirmada: confirmarContrasenna.value
  }
  try {
    const result = await saveDataPronosticoEnviarObjeto(url, payload, dialogLoad)
    await loginDesdeRecuperacion(result.resultado.result.usu, result.resultado.result.cont)
    $q.notify({ type: 'positive', message: 'Contraseña cambiada correctamente ✅' })
    showForgotDialog.value = false
  } catch (err) {
    $q.notify({ type: 'negative', message: 'Error al cambiar la contraseña' })
  }
}

function abrirDialogoRecuperarContrasenha() {
  showForgotDialog.value = true
  forgotEmail.value = ''
  nuevaContrasenna.value = ''
  confirmarContrasenna.value = ''
  correoValido.value = false
}
</script>

<style scoped lang="scss">
@import '../css/quasar.variables.scss';

.login-bg {
  min-height: 100vh;
  display: flex;
  align-items: center;
  justify-content: center;
  background: radial-gradient(
    circle at 50% 30%,
    $bry-white 0%,
    $primary 100%
  );
  padding: 20px;
}

.login-card {
  width: 100%;
  max-width: 350px;
  padding: 40px 32px 32px 32px;
  border-radius: 32px;
  background: rgba(255, 255, 255, 0.08);
  box-shadow: 0 8px 32px 0 rgba(31, 38, 135, 0.37);
  backdrop-filter: blur(8px);
  -webkit-backdrop-filter: blur(8px);
  display: flex;
  flex-direction: column;
  align-items: center;
}

.login-btn {
  background: linear-gradient(90deg, $bry-secondary 0%, $bry-primary 100%);
  color: $bry-white;
  font-weight: bold;
  letter-spacing: 2px;
}

.logo-container {
  cursor: pointer;
  transition: transform 0.3s ease;
  display: flex;
  justify-content: center;
  align-items: center;
}

.logo-container:hover {
  transform: scale(1.05);
}

/* Media Queries para responsividad */
@media (max-width: 599px) {
  .login-card {
    padding: 30px 20px 20px 20px;
    width: 90%;
  }

  .logo-container q-img {
    width: 90px !important;
    height: 90px !important;
  }

  :deep(.q-field) {
    font-size: 14px;
  }

  :deep(.q-input__control) {
    font-size: 14px;
  }
}

@media (max-width: 1023px) and (min-width: 600px) {
  .login-card {
    padding: 35px 28px 28px 28px;
    width: 85%;
  }
}

@media (min-width: 1366px) {
  .login-card {
    width: 400px;
  }
}
</style>
