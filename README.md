# Fase-3_-Simulacion

% 1. CONFIGURACION DEL SISTEMA
horas = 1:24;
capacidad_bateria = 40;  % Capacidad nominal (kWh)
SOC_inicial = 60;        % SOC inicial (%)
% 2. GENERACION SOLAR (kW)
generacion_solar = [0 0 0 0 0 5 8 12 16 20 23 25 ...
                    23 20 16 12 8 5 0 0 0 0 0 0];
% 3. GENERACION EOLICA TOTAL (kW)
generacion_eolica = [8 10 7 12 9 11 6 13 10 8 12 14 ...
                     9 11 15 10 7 12 9 13 8 11 6 10];
% 4. DEMANDA TOTAL DE LA COMUNIDAD (kW)
demanda = [3 2.5 3 3.5 4 6 7 8 8.5 9 8 7.5 ...
           8 9 10 9.5 10 15 15 15 15 4 3.5 3];
% 5. DISTRIBUCION DE LAS CARGAS
% Se utilizan porcentajes ilustrativos porque no se conocen
% las potencias individuales de cada servicio.
% La suma de las tres cargas es igual a la demanda total.
demanda_escuela = 0.40 * demanda;
demanda_bombeo = 0.35 * demanda;
demanda_alumbrado = 0.25 * demanda;
% 6. GENERACION TOTAL Y BALANCE
generacion_total = generacion_solar + generacion_eolica;
% Balance positivo: superavit.
% Balance negativo: deficit.
balance = generacion_total - demanda;
% 7. INICIALIZACION DE LOS RESULTADOS
SOC = zeros(1,24);
SOC(1) = SOC_inicial;
SOC_previo = SOC_inicial;
demanda_atendida = zeros(1,24);
energia_cargada = zeros(1,24);
energia_descargada = zeros(1,24);
energia_no_suministrada = zeros(1,24);
escuela_activa = true(1,24);
bombeo_activo = true(1,24);
alumbrado_activo = true(1,24);
estado = strings(1,24);
% 8. SIMULACION HORA POR HORA
for h = 1:24
    % Recuperar el SOC al inicio de la hora
    if h == 1
        SOC_previo = SOC_inicial;
    else
        SOC_previo = SOC(h-1);
    end
    Pgen = generacion_total(h);
    Pdem = demanda(h);
    % Por defecto, todos los servicios estan activos
    escuela_activa(h) = true;
    bombeo_activo(h) = true;
    alumbrado_activo(h) = true;
    % ESCENARIO A: SUPERAVIT O EQUILIBRIO
    if Pgen >= Pdem
        % Se cubre toda la demanda.
        demanda_atendida(h) = Pdem;
        % Excedente disponible para cargar la bateria
        excedente = Pgen - Pdem;
        % Actualizacion del SOC segun la formula del enunciado
        SOC_calculado = SOC_previo + ...
            (excedente / capacidad_bateria) * 100;
        % Limitar el SOC al 100 %
        SOC(h) = min(SOC_calculado, 100);
        % Energia almacenada en la bateria durante esta hora
        energia_cargada(h) = ...
            (SOC(h) - SOC_previo) / 100 * capacidad_bateria;
        estado(h) = "Superavit";
    % ESCENARIO B: DEFICIT
    else
        % Faltante si se intentara cubrir toda la demanda
        faltante = Pdem - Pgen;
        % SOC hipotetico despues de cubrir el deficit completo
        SOC_estimado = SOC_previo - ...
            (faltante / capacidad_bateria) * 100;
        % Evitar valores negativos para evaluar las prioridades
        SOC_estimado = max(SOC_estimado, 0);
        % ----------------------------------------------------
        % DECISION DE LOAD SHEDDING
        % ----------------------------------------------------
        if SOC_estimado > 40
            % RUTA SEGURA: todos los servicios activos
            escuela_activa(h) = true;
            bombeo_activo(h) = true;
            alumbrado_activo(h) = true;
            estado(h) = "Segura";
        elseif SOC_estimado > 20
            % RUTA DE ALERTA: desconectar alumbrado
            escuela_activa(h) = true;
            bombeo_activo(h) = true;
            alumbrado_activo(h) = false;
            estado(h) = "Alerta";
        else
            % RUTA DE EMERGENCIA: solo escuela activa
            escuela_activa(h) = true;
            bombeo_activo(h) = false;
            alumbrado_activo(h) = false;
            estado(h) = "Emergencia";
        end
        % ----------------------------------------------------
        % DEMANDA DESPUES DE DESCONECTAR LAS CARGAS
        % ----------------------------------------------------
        Pdem_controlada = ...
            demanda_escuela(h) * escuela_activa(h) + ...
            demanda_bombeo(h) * bombeo_activo(h) + ...
            demanda_alumbrado(h) * alumbrado_activo(h);
        demanda_atendida(h) = Pdem_controlada;
        % Deficit real tras aplicar las prioridades
        deficit_controlado = max(Pdem_controlada - Pgen, 0);
        % Energia disponible en la bateria por encima de 0 % SOC
        energia_disponible = ...
            SOC_previo / 100 * capacidad_bateria;
        % Descarga limitada a la energia disponible
        energia_descargada(h) = min( ...
            deficit_controlado, energia_disponible);
        % SOC final tras la descarga real
        SOC(h) = SOC_previo - ...
            (energia_descargada(h) / capacidad_bateria) * 100;
        % Limitar el SOC al intervalo permitido
        SOC(h) = min(max(SOC(h), 0), 100);
        % Energia que no pudo suministrarse
        energia_no_suministrada(h) = ...
            deficit_controlado - energia_descargada(h);
    end
end
% 9. TABLA DE RESULTADOS
resultados = table( ...
    horas', generacion_solar', generacion_eolica', ...
    generacion_total', demanda', balance', demanda_atendida', ...
    energia_cargada', energia_descargada', ...
    energia_no_suministrada', SOC', estado', ...
    escuela_activa', bombeo_activo', alumbrado_activo', ...
    'VariableNames', {'Hora','Solar_kW','Eolica_kW', ...
    'GeneracionTotal_kW','DemandaOriginal_kW','Balance_kW', ...
    'DemandaControlada_kW','CargaBateria_kWh', ...
    'DescargaBateria_kWh','EnergiaNoSuministrada_kWh', ...
    'SOC_Final_porcentaje','Ruta','Escuela','Bombeo','Alumbrado'});
disp('RESULTADOS DE LA SIMULACION');
disp(resultados);
fprintf('SOC inicial: %.2f %%\n', SOC_inicial);
fprintf('SOC final: %.2f %%\n', SOC(end));
fprintf('Energia total cargada: %.2f kWh\n', sum(energia_cargada));
fprintf('Energia total descargada: %.2f kWh\n', sum(energia_descargada));
fprintf('Energia no suministrada: %.2f kWh\n', ...
    sum(energia_no_suministrada));
% 10. GRAFICA DE GENERACION Y DEMANDA
figure;
plot(horas, generacion_total, 'b-o', 'LineWidth', 1.5);
hold on;
plot(horas, demanda, 'r-s', 'LineWidth', 1.5);
plot(horas, demanda_atendida, 'g-d', 'LineWidth', 1.5);
title('Generacion, demanda y control de la micro-red');
xlabel('Tiempo en horas');
ylabel('Potencia (kW)');
legend('Generacion total', 'Demanda original', ...
       'Demanda controlada', 'Location', 'best');
grid on;
xlim([1 24]);
xticks(1:24);
hold off;
% 11. GRAFICA DEL ESTADO DE CARGA
figure;
plot(horas, SOC, 'k-o', 'LineWidth', 1.5);
hold on;
yline(40, '--b', 'Umbral 40 %');
yline(20, '--r', 'Umbral 20 %');
title('Estado de carga de la bateria');
xlabel('Tiempo en horas');
ylabel('SOC (%)');
ylim([0 105]);
xlim([1 24]);
xticks(1:24);
legend('SOC', 'Umbral 40 %', 'Umbral 20 %', ...
       'Location', 'best');
grid on;
hold off;
% 12. GRAFICA DE ENERGIA NO SUMINISTRADA
figure;
bar(horas, energia_no_suministrada);
title('Energia no suministrada por hora');
xlabel('Tiempo en horas');
ylabel('Energia (kWh)');
xlim([0 25]);
xticks(1:24);
grid on;
