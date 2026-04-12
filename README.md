# home_assistant

## ESPHome (JC2432W328): luminosità display automatica con sensore luce

Per aggiungere la gestione automatica della retroilluminazione (backlight) in base al sensore luce già presente sul JC2432W328, inserisci questi blocchi nel tuo YAML ESPHome.

### 1) Substitutions (soglie e limiti)

```yaml
substitutions:
  # ...già esistenti...
  ldr_update_interval: "5s"       # frequenza aggiornamento luminosità
  ldr_dark_voltage: "2.70"        # più alto = più buio (tarare sul tuo pannello)
  ldr_bright_voltage: "0.40"      # più basso = più luce (tarare sul tuo pannello)
  backlight_min: "0.20"           # luminosità minima (0.00-1.00)
  backlight_max: "1.00"           # luminosità massima (0.00-1.00)
```

### 2) Sensore ADC per LDR (pin tipico CYD: GPIO34)

```yaml
sensor:
  # --- Sensore luce onboard (LDR) ---
  - platform: adc
    pin: GPIO34
    id: ambient_light_adc
    name: "${friendly_name} Ambient Light ADC"
    attenuation: auto
    update_interval: ${ldr_update_interval}
    filters:
      - median:
          window_size: 7
          send_every: 1
          send_first_at: 1
      - exponential_moving_average:
          alpha: 0.2
          send_every: 1
    on_value:
      then:
        - lambda: |-
            // Mappa tensione LDR -> luminosità backlight.
            // Su CYD comune: più luce => tensione più bassa.
            const float v_dark = ${ldr_dark_voltage};
            const float v_bright = ${ldr_bright_voltage};

            // Normalizzazione invertita: 0=buio, 1=luce
            float norm = (v_dark - x) / (v_dark - v_bright);
            if (norm < 0.0f) norm = 0.0f;
            if (norm > 1.0f) norm = 1.0f;

            // Limiti luminosità reali (0..1)
            const float min_b = ${backlight_min};
            const float max_b = ${backlight_max};
            float target = min_b + (max_b - min_b) * norm;

            // Evita aggiornamenti troppo frequenti (flicker)
            static float last = -1.0f;
            if (last < 0.0f || fabsf(target - last) >= 0.03f) {
              auto call = id(backlight).turn_on();
              call.set_brightness(target);
              call.set_transition_length(800);
              call.perform();
              last = target;
            }
```

> Se nel tuo modulo il comportamento fosse invertito (più luce = tensione più alta), usa:
> `float norm = (x - v_dark) / (v_bright - v_dark);`

### 3) Mantieni la luce backlight esistente

Il tuo blocco attuale va bene; assicurati solo che `id: backlight` resti invariato:

```yaml
output:
  - platform: ledc
    pin: GPIO27
    id: backlight_pwm

light:
  - platform: monochromatic
    output: backlight_pwm
    name: Display Backlight
    id: backlight
    restore_mode: ALWAYS_ON
    default_transition_length: 0.5s
```

### Note pratiche di taratura

- Se di notte è ancora troppo forte, aumenta `ldr_dark_voltage` o abbassa `backlight_min`.
- Se di giorno è troppo bassa, diminuisci `ldr_bright_voltage`.
- Per debug rapido, osserva il valore del sensore `Ambient Light ADC` in Home Assistant e regola le due soglie.
