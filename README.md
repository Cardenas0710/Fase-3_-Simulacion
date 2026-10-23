# Fase-3_-Simulacion
```
clc; clear; close all; % Limpia la consola, las variables y las figuras
% 1. CONFIGURACION INICIAL
horas = 1:24; % Define las 24 horas del dia
capacidad_bateria = 40; % Capacidad de la bateria en kWh
SOC_inicial = 50; % Estado de carga inicial en porcentaje
SOC = SOC_inicial; % Inicializa el SOC de la bateria
% Distribucion supuesta de la demanda entre los servicios
porcentaje_escuela = 0.40;
porcentaje_bombeo = 0.35;
porcentaje_alumbrado = 0.25;
% 2. VECTORES DE GENERACION Y DEMANDA
% Generacion solar: cero durante la noche
generacion_solar = zeros(1,24);
% Generacion solar creciente durante la manana
generacion_solar(6:11) = [2+rand, 5+2*rand, 8+2*rand, ...
                          12+2*rand, 17+2*rand, 21+2*rand];
% Maxima generacion solar a las 12:00
generacion_solar(12) = 25;
% Generacion solar decreciente durante la tarde
generacion_solar(13:18) = [20+2*rand, 16+2*rand, ...
                           12+2*rand, 8+2*rand, ...
                           4+2*rand, 1+rand];
% Sin generacion solar desde las 19:00 hasta las 24:00
generacion_solar(19:24) = 0;
% Generacion eolica variable entre 0 y 15 kW
generacion_eolica = round(15*rand(1,24), 2);
% Demanda de madrugada: entre 2 y 4 kW
demanda = zeros(1,24);
demanda(1:5) = 2 + 2*rand(1,5);
% Demanda diurna: entre 6 y 10 kW
demanda(6:17) = 6 + 4*rand(1,12);
% Demanda de 18:00 a 21:00: 15 kW
demanda(18:21) = 15;
% Demanda al final del dia: entre 2 y 4 kW
demanda(22:24) = 2 + 2*rand(1,3);
% Redondea los valores a dos decimales
demanda = round(demanda, 2);
% Calcula la generacion total y el balance energetico
generacion_total = generacion_solar + generacion_eolica;
balance = generacion_total - demanda;
% 3. INICIALIZACION DE RESULTADOS
SOC_historial = zeros(1,24);
demanda_controlada = zeros(1,24);
energia_no_suministrada = zeros(1,24);
% Vectores para registrar los estados de los servicios
estado_escuela = strings(1,24);
estado_bombeo = strings(1,24);
estado_alumbrado = strings(1,24);
ruta_control = strings(1,24);
% 4. SIMULACION DEL CONTROL DURANTE 24 HORAS
for h = 1:24
    % Guarda el SOC al inicio de la hora
    SOC_anterior = SOC;
    % Obtiene los valores de la hora actual
    generacion = generacion_total(h);
    demanda_original = demanda(h);
    % Inicialmente todos los servicios estan activos
    escuela = true;
    bombeo = true;
    alumbrado = true;
    energia_no_suministrada(h) = 0;
    % 5. EVALUACION DEL BALANCE ENERGETICO
    if generacion >= demanda_original
        % Hay superavit: se atiende toda la demanda
        excedente = generacion - demanda_original;
        % Calcula cuanto espacio queda en la bateria
        espacio_bateria = (100-SOC_anterior)/100 * capacidad_bateria;
        % Carga la bateria sin superar su capacidad
        energia_cargada = min(excedente, espacio_bateria);
        % Actualiza el estado de carga
        SOC = SOC_anterior + ...
              (energia_cargada/capacidad_bateria)*100;
        SOC = min(100, max(0, SOC));
        % Todos los servicios permanecen activos
        ruta_control(h) = "SUPERAVIT";
        demanda_controlada(h) = demanda_original;
    else
        % Hay deficit frente a la demanda original
        faltante = demanda_original - generacion;
        % Estima el SOC si se atendiera toda la demanda original
        SOC_estimado = SOC_anterior - ...
                       (faltante/capacidad_bateria)*100;
        % 6. SELECCION DE LA RUTA DE CONTROL
        if SOC_estimado > 40
            % Ruta segura: todos los servicios activos
            escuela = true;
            bombeo = true;
            alumbrado = true;
            ruta_control(h) = "SEGURA";
        elseif SOC_estimado > 20
            % Ruta de alerta: se desconecta el alumbrado
            escuela = true;
            bombeo = true;
            alumbrado = false;
            ruta_control(h) = "ALERTA";
        else
            % Ruta de emergencia: solo queda activa la escuela
            escuela = true;
            bombeo = false;
            alumbrado = false;
            ruta_control(h) = "EMERGENCIA";
        end
        % 7. CALCULO DE LA DEMANDA CONTROLADA
        consumo_escuela = demanda_original * porcentaje_escuela;
        consumo_bombeo = demanda_original * porcentaje_bombeo;
        consumo_alumbrado = demanda_original * porcentaje_alumbrado;
        demanda_actual = 0;
        if escuela
            demanda_actual = demanda_actual + consumo_escuela;
        end
        if bombeo
            demanda_actual = demanda_actual + consumo_bombeo;
        end
        if alumbrado
            demanda_actual = demanda_actual + consumo_alumbrado;
        end
        demanda_controlada(h) = demanda_actual;
        % 8. DESCARGA DE LA BATERIA
        % Calcula el deficit despues de aplicar la desconexion
        deficit_controlado = max(0, demanda_actual - generacion);
        % Energia que queda disponible en la bateria
        energia_disponible = (SOC_anterior/100)*capacidad_bateria;
        % La bateria no entrega mas energia de la que almacena
        energia_descargada = min(deficit_controlado, energia_disponible);
        % Actualiza el SOC despues de la descarga
        SOC = SOC_anterior - ...
              (energia_descargada/capacidad_bateria)*100;
        SOC = min(100, max(0, SOC));
        % Registra la energia que no pudo suministrarse
        energia_no_suministrada(h) = ...
            deficit_controlado - energia_descargada;
    end
    % 9. REGISTRO DE LOS RESULTADOS DE LA HORA
    SOC_historial(h) = SOC;
    % Guarda el estado de cada servicio
    if escuela
        estado_escuela(h) = "Activo";
    else
        estado_escuela(h) = "Suspendido";
    end
    if bombeo
        estado_bombeo(h) = "Activo";
    else
        estado_bombeo(h) = "Suspendido";
    end
    if alumbrado
        estado_alumbrado(h) = "Activo";
    else
        estado_alumbrado(h) = "Suspendido";
    end
end
% 10. TABLA GENERAL DE RESULTADOS
% Organiza los resultados en una tabla con una fila por hora
Resultados = table( ...
    horas', ...
    generacion_solar', ...
    generacion_eolica', ...
    generacion_total', ...
    demanda', ...
    balance', ...
    demanda_controlada', ...
    SOC_historial', ...
    ruta_control', ...
    estado_escuela', ...
    estado_bombeo', ...
    estado_alumbrado', ...
    energia_no_suministrada', ...
    'VariableNames', { ...
    'Hora', ...
    'Solar_kW', ...
    'Eolica_kW', ...
    'GeneracionTotal_kW', ...
    'DemandaOriginal_kW', ...
    'Balance_kW', ...
    'DemandaControlada_kW', ...
    'SOC_Porcentaje', ...
    'Ruta', ...
    'Escuela', ...
    'Bombeo', ...
    'Alumbrado', ...
    'EnergiaNoSuministrada_kWh'});
% Muestra la tabla completa en la consola
disp('========== TABLA DE RESULTADOS POR HORA ==========');
disp(Resultados);
% 11. GRAFICA DE GENERACION TOTAL Y DEMANDA
figure;
plot(horas, generacion_total, '-o', 'LineWidth', 1.5);
hold on;
plot(horas, demanda, '-s', 'LineWidth', 1.5);
title('Generacion renovable total vs. demanda');
xlabel('Hora del dia');
ylabel('Potencia (kW)');
legend('Generacion total', 'Demanda original', ...
       'Location', 'best');
grid on;
xlim([1 24]);
xticks(1:24);
% 12. GRAFICA DE GENERACION SOLAR Y EOLICA
figure;
plot(horas, generacion_solar, '-o', 'LineWidth', 1.5);
hold on;
plot(horas, generacion_eolica, '-s', 'LineWidth', 1.5);
title('Generacion solar y eolica por hora');
xlabel('Hora del dia');
ylabel('Potencia (kW)');
legend('Generacion solar', 'Generacion eolica', ...
       'Location', 'best');
grid on;
xlim([1 24]);
xticks(1:24);
% 13. GRAFICA DEL SOC DE LA BATERIA
figure;
plot(horas, SOC_historial, '-o', 'LineWidth', 1.5);
title('Estado de carga de la bateria durante 24 horas');
xlabel('Hora del dia');
ylabel('SOC (%)');
grid on;
xlim([1 24]);
xticks(1:24);
ylim([0 100]);
% 14. RESUMEN FINAL
fprintf('\n========== RESUMEN FINAL ==========\n');
fprintf('SOC inicial: %.2f %%\n', SOC_inicial);
fprintf('SOC final: %.2f %%\n', SOC_historial(24));
fprintf('Energia total no suministrada: %.2f kWh\n', ...
        sum(energia_no_suministrada));
```
