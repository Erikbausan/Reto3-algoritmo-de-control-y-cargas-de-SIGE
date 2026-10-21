# Reto3-algoritmo-de-control-y-cargas-de-SIGE
%% SISTEMA INTELIGENTE DE GESTION ENERGETICA (SIGE) - VERSIÓN FINAL
% Simulación horaria de una micro-red rural híbrida
% Basado en el Reto 2 (Generación y Demanda) + Lógica de Control del documento
clc; clear; close all;

%% 1. CONFIGURACIÓN INICIAL (Vectores de 24 posiciones)
t = 1:24; % Vector de tiempo (24 horas)

% --- Datos de Generación (Reto 2) ---
Pmax_solar = 25;
P_solar = Pmax_solar * sin(pi*(t-5)/14);
P_solar(t <= 5 | t >= 19) = 0; % Sin producción nocturna
P_solar = round(P_solar, 2);

P_eolica = [8 9.5 11 10 7.5 6 5 4.5 6 7 5.5 4 ...
    3.5 5 6.5 8 9 10.5 12 13.5 12 11 9.5 10];
P_eolica = min(P_eolica, 15); % Límite de potencia nominal

E_gen = P_solar + P_eolica; % Generación Híbrida Total [kW]

% --- Datos de Demanda (Reto 2) ---
E_dem = [3 2.5 2 2.5 3.5, ...          % Horas 1-5: Consumo base nocturno
    6 7 8 9 9.5 10 9 8.5 9 9.5 8 7, ... % Horas 6-17: Escuela y bombeo
    12 14 15 13, ...                   % Horas 18-21: Alumbrado y pico
    4 3.5 3];                          % Horas 22-24: Retorno

% --- Variables de la Batería (SOC) ---
Capacidad_bat = 100;   % Capacidad total en kWh
SOC = 50;              % Estado de Carga inicial (50%)
SOC_max = 100;         % Límite máximo
SOC_min = 0;           % Límite mínimo
umbral_alerta = 40;    % Umbral para apagar alumbrado
umbral_emergen = 20;   % Umbral para apagar bombeo

% --- Registro de datos ---
balance_h = zeros(1, 24);
SOC_h = zeros(1, 24);
escuela_h = true(1, 24);
bombeo_h = true(1, 24);
alumbrado_h = true(1, 24);
mode_h = strings(1, 24);

%% 2. BUCLE DE SIMULACIÓN (Evaluación hora por hora)
for h = 1:24
    balance = E_gen(h) - E_dem(h);
    
    % --- LÓGICA DE DECISIÓN AUTOMATIZADA ---
    if E_gen(h) >= E_dem(h)
        % ========== ESCENARIO DE SUPERÁVIT ==========
        Excedente = balance;
        SOC = SOC + (Excedente / Capacidad_bat) * 100;
        SOC = min(SOC, SOC_max); % Limita al 100%
        
        % Todos los servicios activos
        escuela = true; bombeo = true; alumbrado = true;
        mode = "Superávit - Carga";
        
    else
        % ========== ESCENARIO DE DÉFICIT ==========
        Faltante = E_dem(h) - E_gen(h);
        SOC = SOC - (Faltante / Capacidad_bat) * 100;
        SOC = max(SOC, SOC_min); % Limita al 0%
        
        % --- Protocolo de Desconexión de Cargas (Load Shedding) ---
        if SOC > umbral_alerta
            % Ruta Segura
            escuela = true; bombeo = true; alumbrado = true;
            mode = "Déficit - Ruta Segura";
        elseif SOC > umbral_emergen
            % Ruta de Alerta (Apaga Alumbrado)
            escuela = true; bombeo = true; alumbrado = false;
            mode = "Déficit - Ruta Alerta";
        else
            % Ruta de Emergencia (Apaga Alumbrado y Bombeo)
            escuela = true; bombeo = false; alumbrado = false;
            mode = "Déficit - Emergencia";
        end
    end
    
    % --- Almacenar resultados de la hora ---
    balance_h(h) = balance;
    SOC_h(h) = SOC;
    escuela_h(h) = escuela;
    bombeo_h(h) = bombeo;
    alumbrado_h(h) = alumbrado;
    mode_h(h) = mode;
end

%% 3. REPORTE EN CONSOLA (TABLA DE 24 HORAS)
estado = ["SUSPENDIDO", "ACTIVO"];
fprintf('==========================================================================================\n');
fprintf('        REPORTE DEL CONTROLADOR CENTRAL - MICRO-RED HÍBRIDA SIGE (CICLO DE 24 h)\n');
fprintf('==========================================================================================\n');
fprintf('%-5s %-8s %-8s %-9s %-7s %-22s %-10s %-10s %-10s\n', ...
    'Hora','Gen(kW)','Dem(kW)','Bal(kW)','SOC(%)','Mode','Escuela','Bombeo','Alumbrado');
fprintf('------------------------------------------------------------------------------------------\n');
for h = 1:24
    fprintf('%-5d %-8.2f %-8.2f %+-9.2f %-7.2f %-22s %-10s %-10s %-10s\n', ...
        h, E_gen(h), E_dem(h), balance_h(h), SOC_h(h), mode_h(h), ...
        estado(escuela_h(h)+1), estado(bombeo_h(h)+1), estado(alumbrado_h(h)+1));
end
fprintf('------------------------------------------------------------------------------------------\n');
fprintf('SOC final: %.2f %% | SOC mínimo: %.2f %% | Horas en déficit: %d | Horas con cortes: %d\n', ...
    SOC_h(end), min(SOC_h), sum(balance_h < 0), sum(bombeo_h == 0 | alumbrado_h == 0));

%% 4. TABLA DE RESULTADOS
Resultados = table((1:24)', E_gen', E_dem', ...
    max(balance_h', 0), max(-balance_h', 0), SOC_h', escuela_h', ... 
    bombeo_h', alumbrado_h', mode_h', ... 
    'VariableNames', {'Hora','Generacion_kW', ...
    'Demanda_kW','Excedente_kW','Faltante_kW', ...
    'SOC_porcentaje','Escuela_ON','Bombeo_ON', ...
    'Alumbrado_ON','Estado'});

disp('RESULTADOS DEL CONTROLADOR SIGE (TABLA DE 24 HORAS):');
disp(Resultados);

%% 5. GRÁFICAS
% Gráfica 1: Balance y SOC
figure('Name','Balance y SOC - 24 Horas');
yyaxis left
bar(1:24, balance_h, 'FaceColor', [0.2 0.6 1]);
ylabel('Balance energético (kW)');
yyaxis right
plot(1:24, SOC_h, '-o', 'LineWidth', 2, 'Color', 'r');
ylabel('SOC (%)'); ylim([0 105]);
xlabel('Hora del día'); xticks(1:24); grid on;
title('Balance energético y estado de carga de la batería (24 h)');

% Gráfica 2: Generación vs Demanda y SOC
figure('Name','Generación vs Demanda - 24 Horas');

subplot(2,1,1);
plot(1:24, E_gen, '-o', 'LineWidth', 1.5, 'Color', 'b'); hold on;
plot(1:24, E_dem, '-s', 'LineWidth', 1.5, 'Color', 'r'); 
grid on; xlabel('Hora del día'); 
ylabel('Energía (kW)');
title('Generación híbrida frente a demanda (24 h)');
legend('Generación solar + eólica', 'Demanda', 'Location', 'best');
xticks(1:24);

subplot(2,1,2);
plot(1:24, SOC_h, '-o', 'LineWidth', 1.5, 'Color', 'k'); hold on;
yline(umbral_alerta, '--g', 'Alerta 40%');
yline(umbral_emergen, '--r', 'Emergencia 20%');
grid on; xlabel('Hora del día'); ylabel('SOC (%)');
title('Estado de carga de la batería (24 h)');
ylim([0 105]); xticks(1:24);)
