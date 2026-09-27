<div align="center">
  <h1>🎓 UniNotes: Procesamiento Inteligente de Apuntes Universitarios</h1>
  <p><i>Un caso de estudio avanzado sobre <strong>Template Method + Reflection y Metadatos</strong> (Arquitectura en 3 Capas)</i></p>
</div>

<hr/>

## 📖 El Desafío del Cliente

**"UniNotes"** es una plataforma centralizada para que estudiantes de diversas universidades compartan sus apuntes. La mecánica: un alumno sube sus apuntes proporcionando un enlace de **Google Drive** y selecciona su Universidad.

El flujo conceptual de procesamiento es idéntico para todos:
1. **Descargar** el archivo.
2. **Validar** formato y contenido.
3. **Extraer Metadatos** (Materia).
4. **Almacenar**.

**El problema:** Cada universidad (UBA, UTN, Siglo 21, etc.) tiene reglas diferentes de validación y extracción. 

---

## 🛠️ La Solución : Template Method + Reflection (Capas)

Implementamos una arquitectura robusta en **N-Capas** (UI, BLL, BE, Servicios) utilizando el patrón **Template Method** en la capa de `Servicios` para estandarizar el algoritmo. Adicionalmente, aplicamos **Reflection y Atributos Personalizados (Metadatos)** en la capa `BLL` para descubrir instanciar el procesador correcto en tiempo de ejecución de manera dinámica, eliminando por completo los anti-patrones de múltiples `if/else` o `switch`.

### 1. El Metadato (Custom Attribute)
Se crea un atributo para decorar las clases que procesan las notas.
```csharp
[AttributeUsage(AttributeTargets.Class, Inherited = false)]
public class UniversidadProcesadorAttribute : Attribute
{
    public string NombreUniversidad { get; }
    public UniversidadProcesadorAttribute(string nombreUniversidad) => NombreUniversidad = nombreUniversidad;
}
```

### 2. El Template Method y los Procesadores Decorados
```csharp
public abstract class ProcesadorApunteTemplate
{
    public Apunte Procesar(Apunte apunte)
    {
        DescargarDesdeDrive(apunte);
        if(ValidarApunte(apunte)) { ExtraerMetadatos(apunte); apunte.EstaValidado = true; }
        return apunte;
    }
    protected abstract bool ValidarApunte(Apunte apunte);
    protected abstract void ExtraerMetadatos(Apunte apunte);
}

[UniversidadProcesador("UBA")]
public class ProcesadorUBA : ProcesadorApunteTemplate { /* Implementación UBA */ }

[UniversidadProcesador("UTN")]
public class ProcesadorUTN : ProcesadorApunteTemplate { /* Implementación UTN */ }
```

### 3. Factory Dinámico con Reflection en BLL
El Gestor ya no usa un `switch`. Escanea el *Assembly*, busca la clase hija del Template que contenga el `UniversidadProcesadorAttribute` correspondiente, y la instancia de forma mágica.
```csharp
private ProcesadorApunteTemplate ObtenerProcesador(string universidad)
{
    var assembly = typeof(ProcesadorApunteTemplate).Assembly;

    var tipoProcesador = assembly.GetTypes()
        .Where(t => t.IsClass && !t.IsAbstract && t.IsSubclassOf(typeof(ProcesadorApunteTemplate)))
        .FirstOrDefault(t => 
        {
            var atributo = t.GetCustomAttribute<UniversidadProcesadorAttribute>();
            return atributo != null && atributo.NombreUniversidad.Equals(universidad, StringComparison.OrdinalIgnoreCase);
        });

    if (tipoProcesador != null)
        return (ProcesadorApunteTemplate)Activator.CreateInstance(tipoProcesador);

    return null;
}
```

## 🎯 Beneficios de este Enfoque (Nivel Senior)
1. **Verdadero Principio Abierto/Cerrado (OCP):** Agregar una nueva universidad (`ProcesadorSiglo21`) ahora solo requiere crear la clase y ponerle el atributo `[UniversidadProcesador("Siglo21")]`. ¡No hay que tocar la `BLL` en lo absoluto!
2. **Descubrimiento Dinámico:** La instanciación delegada a `Reflection` reduce el acoplamiento y promueve arquitecturas modulares estilo *Plug & Play*.
