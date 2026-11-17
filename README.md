<div align="center">

<img width="1200" height="475" alt="GHBanner" src="https://github.com/user-attachments/assets/0aa67016-6eaf-458a-adb2-6e31a0763ed6" />

  <h1>Built with AI Studio</h2>

  <p>The fastest path from prompt to production with Gemini.</p>

  <a href="https://aistudio.google.com/apps">Start building</a>

</div>
# Diagrama de arquitectura y componentes

```
┌────────────────────────────────────────────────────────────────────────┐
│ Actividad principal (Actividad de fragmento) │
│ De extremo a extremo · Host de BiometricPrompt · POST_NOTIFICATIONS · Puerta de autenticación │
└─────────────────────────────────────┬────────────────────────────────────┘
                                      │
                  ┌───────────────────┴───────────────────┐
                  ▼ ▼
┌────────────────────────────────┐ ┌───────────────────────────────────┐
│ AuthScreen (Sin autenticación) │ │ HashPulseApp (Autenticado) │
│ CredentialManager Autenticación de Google │ │ Clave por user.uid · Host de navegación │
└────────────────────────────────┘ └─────────────────┬────────────────┘
                                                          │
         ┌───────────────────────┬──────────────────────┼──────────────────────┐
         ▼ ▼ ▼ ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│ Pantalla del panel de control │ │ Pantalla de equipos │ │ Pantalla de la cartera │ │ Configuración del historial │
│ HashChart · KPI │ │ Agregar/Alternar Rigs │ │ Retirar · Comisiones │ │ Ledger · Alertas │
└────────┬────────┘ └────────┬───────┘ └────────┬───────┘ └────────┬───────┘
         │ │ │ │
         └───────────────────────┴───────────┬───────────┴──────────────────────┘
                                              ▼
                       ┌────────────────────────────────────────────┐
                       │ ViewModels (StateFlow / UiState) │
                       │ DashboardVM · WalletVM · SettingsVM │
                       └─────────────────────┬──────────────────────┘
                                              │
           ┌─────────────────────────────────┼────────────────────────────────┐
           ▼ ▼ ▼
┌───────────────────────┐ ┌───────────────────────────┐ ┌────────────────────────┐
│ MempoolApiService │ │ FirestoreMiningRepo │ │ Seguridad y alertas │
│ Retrofit + Moshi (BTC │ │ Billetera · Rigs · Ledger │ │ BiometricAuthHelper │
│ Precio/Tasa de hash/Comisiones) │ │ Transacciones y oyentes │ │ BtcAddressValidator │
└────────────────────────┘ └─────────────┬─────────────┘ │ PayoutNotifier │
                                              │ └────────────────────────┘
                                              ▼
                       ┌────────────────────────────────────────────┐
                       │ Firebase Firestore (ID de base de datos con nombre) │
                       │ /users/{uid}/wallet/primary │
                       │ /users/{uid}/rigs/{rigId} │
                       │ /users/{uid}/transactions/{txId} │
                       │ /users/{uid}/savedAddresses/{addrId} │
                       └────────────────────────────────────────────┘
```

### Modelo de datos y estado

Todos los datos propiedad del usuario se encuentran en `/users/{userId}/...` en la base de datos Firestore con nombre (`R.string.firestore_database_id`) y están estrictamente aislados por `request.auth.uid == userId` con validación de esquema a nivel de campo en `firestore.rules`:

- **`WalletProfile`** (`/users/{userId}/wallet/primary`):
  - `userId: String`, `availableSats: Long`, `pendingSats: Long`, `totalMinedSats: Long`, `payoutThresholdSats: Long`, `alertsEnabled: Boolean`, `updatedAt: Timestamp?`.
- **`MiningRig`** (`/users/{userId}/rigs/{rigId}`):
  - `@DocumentId id: String`, `userId: String`, `name: String`, `model: String`, `nominalThs: Double`, `powerWatts: Long`, `powerMode: String` (`ECO` | `BALANCED` | `OVERCLOCK`), `status: String` (`ACTIVE` | `STANDBY`), `temperatureC: Double`, `efficiencyPercent: Double`, `createdAt: Timestamp?`, `updatedAt: Timestamp?`.
- **`WalletTransaction`** (`/users/{userId}/transactions/{txId}`):
  - `@DocumentId id: String`, `userId: String`, `type: String` (`MINING_REWARD` | `WITHDRAWAL`), `amountSats: Long`, `feeSats: Long`, `usdValueAtTime: Double`, `destinationAddress: String?`, `txHashPreview: String`, `blockHeight: Long`, `status: String` (`CONFIRMED` | `BROADCASTING`), `biometricVerified: Boolean`, `createdAt: Timestamp?`.
- **`SavedAddress`** (`/users/{userId}/savedAddresses/{addrId}`):
  - `@DocumentId id: String`, `userId: String`, `label: String`, `address: String`, `networkType: String` (`BECH32` | `LEGACY`), `createdAt: Timestamp?`.
- **`NetworkTelemetry`** (estado en memoria actualizado desde `mempool.space`):
  - `btcPriceUsd: Double`, `networkHashrateEh: Double`, `tipBlockHeight: Long`, `difficultyChangePercent: Double`, `difficultyProgressPercent: Double`, `fastestFeeSatVb: Int`, `halfHourFeeSatVb: Int`, `economyFeeSatVb: Int`, `hashrateHistoryPoints: List<HashratePoint>`.

### Componente interactivo y mapeo de estados

- **Indicador de tasa de hash en vivo y gráfico Canvas**: Un indicador de corrutina en `DashboardViewModel` muestrea los equipos Firestore activos cada 2 segundos, aplicando una variación térmica/de hash realista alrededor del `TH/s` nominal de cada equipo y el multiplicador `powerMode` (`0.82x` Eco, `1.0x` Balanced, `1.14x` Overclock), actualizando el gráfico Canvas y calculando los satoshis pendientes en vivo por minuto en función de la dificultad actual de la red.
- **Liquidar / Reclamar recompensas de minería**: Al tocar **Reclamar recompensas pendientes** se ejecuta una transacción `runTransaction` de Firestore que mueve los `pendingSats` acumulados a `availableSats`, incrementa `totalMinedSats`, registra un documento de libro mayor `MINING_REWARD` sellado con `FieldValue.serverTimestamp()` y la altura actual de `tipBlockHeight`, y activa `PayoutNotifier` si `alertsEnabled` es verdadero.
- **Acciones de control de la plataforma**: Cambiar el interruptor de una plataforma (`ACTIVO` / `EN ESPERA`), cambiar el perfil de energía o agregar un nuevo minero en la nube llama a `FirestoreMiningRepo` devolviendo `Result<Unit>`, actualizando la interfaz de usuario de forma optimista a través de los oyentes de instantáneas y mostrando los errores de forma clara en una `Snackbar`.
- **Proceso de retirada biométrica**:
  1. `WalletViewModel` valida `amountSats > 0`, `amountSats + feeSats <= availableSats` y `BtcAddressValidator.isValid(address)` en tiempo real.
  2. Al pulsar **Autorizar y retirar** se inicia `BiometricAuthHelper.authenticate(fragmentActivity)` (`BIOMETRIC_STRONG o DEVICE_CREDENTIAL`).
  3. Tras una autenticación exitosa, `FirestoreMiningRepo.submitWithdrawal(...)` ejecuta una transacción atómica de Firestore deduciendo `amountSats + feeSats`, creando un registro `WITHDRAWAL` (`biometricVerified = true`) y activando una notificación de alerta de pago de alta prioridad.
- **Permiso de notificación y control deslizante de umbral**: Activar o desactivar las **alertas push de pago** en Configuración solicita `Manifest.permission.POST_NOTIFICATIONS` en Android 13+ a través de `ActivityResultContracts.RequestPermission()`, y las preferencias de alerta y los umbrales se guardan en `/users/{uid}/wallet/primary`.
