# Practica-Microfono-y-Camara-II

Autor: Eric Bermúdez Hernández

Email: alu0101517476@ull.edu.es

---

## 1. Introducción

En esta práctica se parte de una escena previa que contiene varios guerreros que reaccionaban al tocar determinados objetivos (shields). El objetivo principal es extender esa escena incorporando funcionalidades multimedia mediante el uso del micrófono y la cámara web del ordenador, además de completar la lógica para que los objetivos reproduzcan sonidos al ser recogidos.

La práctica se divide en dos grandes bloques:

1. Escena 1 – Guerreros + Objetivos (Shields)

- Añadir sonido al recoger los objetivos.

- Mantener la lógica de movimiento y recolección ya existente.

2. Escena 2 – Micrófono y Reproducción en Tiempo Real

- Crear una nueva escena con altavoces/pantalla que reproduzcan lo captado por el micrófono.

- Control por teclado (R para grabar/detener).

3. Escena 3 – Cámara Web (TV)

- Mostrar la webcam en una pantalla de la escena.

- Control por teclado (S para arrancar, P para parar, X para capturar una imagen).

- Guardar capturas como PNG.

---

## 2. Parte 1 — Sonido al recoger objetivos

### 2.1 Configuración del objetivo (Shield)

Cada objetivo contiene el script proporcionado `Shield.cs`.
Se modificó para añadir una funcionalidad de audio robusta que funcione incluso al desactivar el objeto.

El Script `Shield.cs` es el siguiente:

```C#
using UnityEngine;

[RequireComponent(typeof(Collider))]
[RequireComponent(typeof(Rigidbody))]
public class Shield : MonoBehaviour
{
    public ShieldType type = ShieldType.Type1;

    public int basePoints = 0;
    public bool isSpecialType1 = false;
    public bool collectOnTouch = true;

    public ShieldMover mover;

    [Header("Audio")]
    public AudioClip collectedSound;

    private bool _collected;

    void Awake()
    {
        if (mover == null) mover = GetComponent<ShieldMover>();
    }

    void Reset()
    {
        var rb = GetComponent<Rigidbody>();
        rb.isKinematic = false;
        rb.useGravity = false;
        rb.collisionDetectionMode = CollisionDetectionMode.Continuous;
        rb.interpolation = RigidbodyInterpolation.Interpolate;
        rb.freezeRotation = true;
    }

    public void Collect()
    {
        if (_collected) return;
        _collected = true;

        if (collectedSound != null)
        {
            AudioSource.PlayClipAtPoint(collectedSound, transform.position);
        }

        gameObject.SetActive(false);
    }
}

```

### 2.2 Configuración del collider del objetivo

- `Collider`: IsTrigger = true

- `Rigidbody`:

  - Use Gravity = false

  - Is Kinematic = false (manejado por el propio script del Shield)

- Campo `Collected Sound`: asignar el audio deseado

--- 

## 3. Parte 1 — Detección por parte del guerrero

Los guerreros deben detectar cuándo tocan un objetivo.
Para ello se añadió un script llamado `WarriorCollector.cs` el cual tiene el siguiente código:

```C#
using UnityEngine;

public class WarriorCollector : MonoBehaviour
{
    private void OnTriggerEnter(Collider other)
    {
        Shield shield = other.GetComponent<Shield>();
        if (shield != null)
        {
            shield.Collect();
        }
    }
}

```

### 3.1 Configuración del guerrero

- `Collider`: IsTrigger = false

- `Rigidbody`:

  - Is Kinematic = true
    (el guerrero se mueve sin física, mediante el inspector o scripts)

- Script añadido: WarriorCollector

Con estos ajustes, al tocar un objetivo, se reproduce el sonido y el shield desaparece.

---

## 4. Parte 2 — Escena del micrófono

Se creó una nueva escena (“MicScene”) formada por:

- Un plano o cubo como pantalla/altavoz llamado Pantalla

- Un `AudioSource`

- Un nuevo script llamado `Recorder.cs`

El objetivo es reproducir por los altavoces lo que detecta el micrófono al pulsar R.

### 4.1 Script Recorder.cs completo

El Script tiene el siguiente código:

```C#
using UnityEngine;

public class Recorder : MonoBehaviour
{
    private AudioSource audioSource;
    private string micDevice;
    private bool isRecording = false;

    public int recordingLength = 10;
    public int frequency = 44100;

    void Start()
    {
        audioSource = GetComponent<AudioSource>();

        if (Microphone.devices.Length > 0)
        {
            micDevice = Microphone.devices[0];
            Debug.Log("Usando micrófono: " + micDevice);
        }
        else
        {
            Debug.LogWarning("No hay micrófonos disponibles.");
        }
    }

    void Update()
    {
        if (Input.GetKeyDown(KeyCode.R))
        {
            if (!isRecording) StartRecording();
            else StopRecording();
        }
    }

    void StartRecording()
    {
        if (micDevice == null) return;

        AudioClip clip = Microphone.Start(micDevice, true, recordingLength, frequency);

        while (Microphone.GetPosition(micDevice) <= 0) { }

        audioSource.clip = clip;
        audioSource.loop = true;
        audioSource.Play();

        isRecording = true;
    }

    void StopRecording()
    {
        Microphone.End(micDevice);
        audioSource.Stop();
        isRecording = false;
    }
}

```

### 4.2 Configuración del objeto “Pantalla”

- Añadir componente AudioSource

  - Play On Awake = false

  - Loop = false

- Añadir script Recorder

--- 

## 5. Parte 3 — Integración de la cámara web

Para la parte final del guion, se reutiliza la escena original de los guerreros (o cualquier otra) y se añade una pantalla que mostrará lo que ve la cámara web.

### 5.1 Preparación de la pantalla

1. Crear un Quad o Cube llamado `TVScreen`.

2. Crear un Material llamado `TVMaterial`.

3. Asignar ese material al `TVScreen`.

---

## 6. Script `TV.cs` para controlar la cámara web

El script permite:

- Pulsar S → iniciar la webcam y mostrarla en pantalla

- Pulsar P → detener la webcam

- Pulsar X → capturar una imagen en PNG

### 6.1 Código completo

```C#
using UnityEngine;
using System.IO;

public class TV : MonoBehaviour
{
    private Material tvMaterial;
    private WebCamTexture webcamTexture;

    public string savePath = "Captures/";
    private int captureCounter = 1;

    void Start()
    {
        Renderer renderer = GetComponent<Renderer>();
        tvMaterial = renderer.material;

        if (WebCamTexture.devices.Length > 0)
        {
            WebCamDevice cam = WebCamTexture.devices[0];
            Debug.Log("Usando cámara: " + cam.name);

            webcamTexture = new WebCamTexture(cam.name);
        }
        else
        {
            Debug.LogWarning("No hay cámaras disponibles.");
        }

        string fullPath = Path.Combine(Application.dataPath, savePath);
        if (!Directory.Exists(fullPath))
            Directory.CreateDirectory(fullPath);
    }

    void Update()
    {
        if (Input.GetKeyDown(KeyCode.S)) StartCamera();
        if (Input.GetKeyDown(KeyCode.P)) StopCamera();
        if (Input.GetKeyDown(KeyCode.X)) TakeSnapshot();
    }

    void StartCamera()
    {
        if (webcamTexture == null) return;

        tvMaterial.mainTexture = webcamTexture;
        webcamTexture.Play();

        Debug.Log("Cámara iniciada.");
    }

    void StopCamera()
    {
        if (webcamTexture != null && webcamTexture.isPlaying)
            webcamTexture.Stop();
    }

    void TakeSnapshot()
    {
        if (webcamTexture == null || !webcamTexture.isPlaying) return;

        Texture2D snapshot = new Texture2D(webcamTexture.width, webcamTexture.height);
        snapshot.SetPixels(webcamTexture.GetPixels());
        snapshot.Apply();

        string fullPath = Path.Combine(Application.dataPath, savePath);
        string fileName = "capture_" + captureCounter.ToString("D3") + ".png";

        File.WriteAllBytes(Path.Combine(fullPath, fileName), snapshot.EncodeToPNG());

        Debug.Log("Captura guardada.");
        captureCounter++;
    }
}

```

### 6.2 Configuración del objeto TVScreen

- Renderer → asignar `TVMaterial`
- Add Component → script `TV.cs`

---

## 7. Configuración del sistema de Input

Unity generaba errores al usar `Input.GetKeyDown` porque el proyecto tenía activado únicamente el Nuevo Input System.

Para solucionarlo:

1. Edit → Project Settings

2. Player → Other Settings

3. Active Input Handling → Both

Esto permite usar la API clásica (`Input.GetKeyDown`), necesaria para esta práctica.

---

## 8. Conclusión

Al finalizar todos los pasos:

- Los guerreros pueden recoger objetivos y reproducir un sonido automáticamente.

- La nueva escena con la pantalla "Pantalla" reproduce en tiempo real el audio del micrófono al pulsar R.

- La escena de los guerreros puede mostrar vídeo en tiempo real desde la cámara web al pulsar S, detenerlo con P y capturar imágenes con X.

- Se implementaron todos los requisitos del guion de la práctica:

  - Uso del micrófono (Microphone.Start)

  - Uso de la cámara (WebCamTexture)

  - Control por teclado

  - Guardado de capturas

  - Integración audiovisual completa con la escena

A continuación se muestra un vídeo de la ejecución de la práctica: