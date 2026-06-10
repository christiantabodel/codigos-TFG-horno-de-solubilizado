codigo para escribir

```matlab
% CURVAS DEL VENTILADOR
% Ajuste polinómico (grado 2) y figuras de la curva del ventilador: escalada
% al 100 % y calibrada al 60 % (la usada en Fluent). Datos: VENTILADORES.xlsx.

clear; clc; close all;

% Asegura que las figuras se abran como ventanas flotantes (no acopladas).
set(groot, 'defaultFigureWindowStyle', 'normal');

% 1.CONFIGURACIÓN

cfg.archivoExcel = 'VENTILADORES.xlsx';                 % nombre del Excel
cfg.hoja         = 'ESCALADO DE VENTILADORES V2';       % hoja con datos buenos
cfg.usarExcel    = true;   % true: lee del Excel | false: usa datos de respaldo


cfg.etqVel = "v[m/s]";            % fila de velocidad normal v  (eje X)
cfg.etq100 = "Ht'[Pa]";          % fila de salto de presión reescalado 100 %
cfg.etq60  = "Ht escalado al 60"; % fila de salto de presión calibrado 60 %
cfg.colDatos = 3:5;               % columnas C,D,E -> los 3 puntos

% Valores de respaldo (del handoff)
% Se emplean si cfg.usarExcel = false o si no se encuentra el archivo Excel.
fallback.v     = [1.310927435, 1.787628321, 2.025978764];   % [m/s]
fallback.dP100 = [705.661390 , 656.707742 , 622.307881 ];   % [Pa]  100 %
fallback.dP60  = [423.396834 , 394.024645 , 373.384729 ];   % [Pa]   60 %

% Ajuste y representación
cfg.gradoPoli    = 2;     % grado del polinomio (el handoff fija grado 2)
cfg.xMin         = 0;     % límite inferior del eje X [m/s]
cfg.xMax         = 2.5;   % límite superior del eje X [m/s]
cfg.nEval        = 220;   % nº de puntos para dibujar curvas suaves
cfg.margenSolido = 0.15;  % [m/s] el tramo CONTINUO cubre los puntos + este
                          % margen a cada lado; el resto del polinomio se
                          % dibuja a trazos (discontinuo = extrapolación).
cfg.mostrarTitulo = false; % false = SIN título ni subtítulo.

% Exportación (opcional; DESACTIVADA por defecto)
% Por defecto no se guarda ningún archivo: las figuras solo se muestran en
% pantalla
cfg.exportar      = false;
cfg.carpetaSalida = 'figuras_ventiladores';
cfg.dpiPNG        = 300;
cfg.exportarSVG   = true;

% Estilo gráfico
st.c100 = [0.00 0.45 0.74];   % azul  -> curva 100 %
st.c60  = [0.85 0.33 0.10];   % naranja-> curva 60 %
st.LW   = 1.8;   % grosor de línea
st.MS   = 8;     % tamaño de marcador
st.FS   = 12;    % tamaño de fuente base
st.FSlab= 13;    % tamaño de fuente de los ejes
st.FSann= 10;    % tamaño de fuente de las anotaciones (ecuaciones)

% Símbolos por código Unicode
SYM.delta = char(916);   % Δ
SYM.sup2  = char(178);   % ²

% 2.CARGA DE DATOS

rutaBase = fileparts(mfilename('fullpath'));
if isempty(rutaBase); rutaBase = pwd; end

leido = false;
if cfg.usarExcel
    rutaExcel = fullfile(rutaBase, cfg.archivoExcel);
    if exist('readcell','file') ~= 2
        warning('readcell no disponible (MATLAB < R2019a). Se usan datos de respaldo.');
    elseif ~isfile(rutaExcel)
        warning('No se encontró "%s". Se usan datos de respaldo.', rutaExcel);
    else
        C    = readcell(rutaExcel, 'Sheet', cfg.hoja);
        v    = leerFila(C, cfg.etqVel, cfg.colDatos);
        dP100= leerFila(C, cfg.etq100, cfg.colDatos);
        dP60 = leerFila(C, cfg.etq60 , cfg.colDatos);
        leido= true;
        fprintf('Datos leídos del Excel: %s (hoja "%s").\n', cfg.archivoExcel, cfg.hoja);
    end
end
if ~leido
    v = fallback.v;  dP100 = fallback.dP100;  dP60 = fallback.dP60;
    fprintf('Usando valores de respaldo del handoff.\n');
end

% 3.AJUSTE POLINÓMICO  (mínimos cuadrados, grado 2)
% polyfit resuelve por mínimos cuadrados; con 3 puntos y grado 2 el ajuste
% es exacto (la parábola pasa por los 3 puntos), de ahí que R² = 1.

[p100, R2_100] = ajustePoli(v, dP100, cfg.gradoPoli);
[p60 , R2_60 ] = ajustePoli(v, dP60 , cfg.gradoPoli);

% Curvas suaves para representar (evaluación del polinomio en el rango X).
xx    = linspace(cfg.xMin, cfg.xMax, cfg.nEval);
yy100 = polyval(p100, xx);
yy60  = polyval(p60 , xx);

% Verificación por consola (debe coincidir con el Excel; R² ≈ 1).
fprintf('Ajuste 100%%: dP = %.4f v^2 %+.4f v %+.4f   (R^2 = %.6f)\n', p100(1),p100(2),p100(3),R2_100);
fprintf('Ajuste  60%%: dP = %.4f v^2 %+.4f v %+.4f   (R^2 = %.6f)\n', p60(1) ,p60(2) ,p60(3) ,R2_60 );

% 4.GENERACIÓN DE FIGURAS
% Tramo CONTINUO: cubre los puntos de datos + un margen (cfg.margenSolido).
% Resto del polinomio: línea DISCONTINUA (extrapolación fuera de los datos).

a  = max(cfg.xMin, min(v) - cfg.margenSolido);   % inicio del tramo continuo
b  = min(cfg.xMax, max(v) + cfg.margenSolido);   % fin    del tramo continuo
iS = (xx >= a) & (xx <= b);                       % máscara del tramo continuo

% Etiquetas de los ejes (con Δ por Unicode).
labX = 'Velocidad normal, v [m/s]';
labY = sprintf('Salto de presión, %sP [Pa]', SYM.delta);

% FIGURA A: 100 % vs 60 %
figA = figure('Color','w','Units','centimeters','Position',[2 2 16 11], ...
              'WindowStyle','normal','Name','Figura A — escalada vs calibrada');
axA  = axes(figA); hold(axA,'on'); grid(axA,'on'); box(axA,'on');
set(axA,'FontSize',st.FS,'GridAlpha',0.15);
axA.XMinorGrid = 'on'; axA.YMinorGrid = 'on'; axA.MinorGridAlpha = 0.07;

plot(axA, xx, yy100, '--','Color',st.c100,'LineWidth',st.LW,'HandleVisibility','off');
plot(axA, xx, yy60 , '--','Color',st.c60 ,'LineWidth',st.LW,'HandleVisibility','off');
hA100 = plot(axA, xx(iS), yy100(iS), '-','Color',st.c100,'LineWidth',st.LW);
hA60  = plot(axA, xx(iS), yy60(iS) , '-','Color',st.c60 ,'LineWidth',st.LW);
% Puntos de ajuste (marcadores distintos por serie).
plot(axA, v, dP100, 'o','MarkerSize',st.MS,'MarkerFaceColor',st.c100,'MarkerEdgeColor','k','LineWidth',0.5,'HandleVisibility','off');
plot(axA, v, dP60 , 's','MarkerSize',st.MS,'MarkerFaceColor',st.c60 ,'MarkerEdgeColor','k','LineWidth',0.5,'HandleVisibility','off');

% Límites, etiquetas y título.
xlim(axA,[cfg.xMin cfg.xMax]);
ylim(axA,[min(yy60)-30, max(yy100)+70]);   % techo ampliado: deja hueco a la leyenda
xlabel(axA,labX,'FontSize',st.FSlab);
ylabel(axA,labY,'FontSize',st.FSlab);
if cfg.mostrarTitulo
    title(axA,'Curva característica del ventilador','FontSize',st.FS+1);
    try, subtitle(axA,'Escalada (100 %) frente a calibrada al 60 % (curva de Fluent)', ...
            'FontSize',st.FS-1,'Color',[0.3 0.3 0.3]); catch, end
end

% Leyenda breve (solo las dos curvas; el color identifica también su ecuación).
legend([hA100 hA60], {'Escalada (100 %)','Calibrada (60 %)'}, ...
       'Location','northeast','FontSize',st.FS,'Box','on');

% Ecuaciones + R². Recuadro blanco para que se lean bien sobre la rejilla. Coma decimal.
eq100 = comaDecimal(sprintf('%sP = %.3f v%s %+.3f v %+.2f   (R%s = %.3f)', ...
        SYM.delta, p100(1), SYM.sup2, p100(2), p100(3), SYM.sup2, R2_100));
eq60  = comaDecimal(sprintf('%sP = %.3f v%s %+.3f v %+.2f   (R%s = %.3f)', ...
        SYM.delta, p60(1) , SYM.sup2, p60(2) , p60(3) , SYM.sup2, R2_60 ));
ylA  = ylim(axA);  rngA = ylA(2)-ylA(1);    % posición relativa a los límites Y
xann = cfg.xMin + 0.05*(cfg.xMax-cfg.xMin);
text(axA, xann, ylA(1)+0.68*rngA, eq100, 'Color',st.c100,'FontSize',st.FSann, ...
     'FontWeight','bold','Interpreter','tex','BackgroundColor','w', ...
     'EdgeColor',st.c100,'Margin',4,'VerticalAlignment','middle');
text(axA, xann, ylA(1)+0.42*rngA, eq60 , 'Color',st.c60 ,'FontSize',st.FSann, ...
     'FontWeight','bold','Interpreter','tex','BackgroundColor','w', ...
     'EdgeColor',st.c60 ,'Margin',4,'VerticalAlignment','middle');

% FIGURA B: solo curva final 60 %
figB = figure('Color','w','Units','centimeters','Position',[2 2 16 11], ...
              'WindowStyle','normal','Name','Figura B — curva final Fluent');
axB  = axes(figB); hold(axB,'on'); grid(axB,'on'); box(axB,'on');
set(axB,'FontSize',st.FS,'GridAlpha',0.15);
axB.XMinorGrid = 'on'; axB.YMinorGrid = 'on'; axB.MinorGridAlpha = 0.07;

plot(axB, xx, yy60, '--','Color',st.c60,'LineWidth',st.LW,'HandleVisibility','off');
hB = plot(axB, xx(iS), yy60(iS), '-','Color',st.c60,'LineWidth',st.LW);
plot(axB, v, dP60, 's','MarkerSize',st.MS,'MarkerFaceColor',st.c60,'MarkerEdgeColor','k','LineWidth',0.5,'HandleVisibility','off');

xlim(axB,[cfg.xMin cfg.xMax]);
ylim(axB,[min(yy60)-20, max(yy60)+70]);
xlabel(axB,labX,'FontSize',st.FSlab);
ylabel(axB,labY,'FontSize',st.FSlab);
if cfg.mostrarTitulo
    title(axB,'Curva del ventilador empleada en Fluent','FontSize',st.FS+1);
    try, subtitle(axB,'Calibrada al 60 % y ajuste polinómico de 2.º grado', ...
            'FontSize',st.FS-1,'Color',[0.3 0.3 0.3]); catch, end
end

legend(hB, {'Calibrada (60 %)'}, 'Location','northeast','FontSize',st.FS,'Box','on');

% Ecuación + R² en recuadro blanco, zona limpia superior izquierda. Coma decimal.
ylB = ylim(axB);
eqB = comaDecimal(sprintf('%sP = %.3f v%s %+.3f v %+.2f\nR%s = %.3f', ...
      SYM.delta, p60(1), SYM.sup2, p60(2), p60(3), SYM.sup2, R2_60));
text(axB, cfg.xMin+0.06*(cfg.xMax-cfg.xMin), ylB(2)-0.08*diff(ylB), eqB, ...
     'Color',st.c60,'FontSize',st.FSann+0.5,'FontWeight','bold','Interpreter','tex', ...
     'BackgroundColor','w','EdgeColor',st.c60,'Margin',4,'VerticalAlignment','top');

% 5.EXPORTACIÓN (opcional; desactivada por defecto)

if cfg.exportar
    carpeta = fullfile(rutaBase, cfg.carpetaSalida);
    if ~exist(carpeta,'dir'); mkdir(carpeta); end
    exportarFigura(figA, fullfile(carpeta,'fan_curves_rescaled_vs_final'), cfg);
    exportarFigura(figB, fullfile(carpeta,'fan_curve_final_fluent')     , cfg);
    fprintf('Figuras exportadas en: %s\n', carpeta);
else
    fprintf('Figuras mostradas en pantalla (no se ha guardado ningún archivo).\n');
end

% FUNCIONES LOCALES

function vec = leerFila(C, etiqueta, colDatos)
%LEERFILA  Devuelve los valores numéricos de la fila cuya etiqueta (col. B)
%          coincide parcialmente con 'etiqueta'.
    objetivo = normaliza(etiqueta);
    fila = 0;
    for r = 1:size(C,1)
        if size(C,2) >= 2
            txt = normaliza(C{r,2});         % etiqueta en la columna B
            if ~isempty(txt) && contains(txt, objetivo)
                fila = r; break;
            end
        end
    end
    if fila == 0
        error(['No se encontró ninguna fila cuya etiqueta (columna B) contenga "%s". ' ...
               'Revisa cfg.etq* o cfg.hoja.'], char(etiqueta));
    end
    crudo = C(fila, colDatos);
    vec   = zeros(1, numel(crudo));
    for k = 1:numel(crudo)
        val = crudo{k};
        if isnumeric(val) && isscalar(val) && ~isnan(val)
            vec(k) = val;
        else
            d = str2double(string(val));
            if isnan(d)
                error('Valor no numérico en la fila "%s", columna %d.', char(etiqueta), colDatos(k));
            end
            vec(k) = d;
        end
    end
end

function s = comaDecimal(s)
%COMADECIMAL  Sustituye el punto decimal por coma.
%   Solo se usa para los TEXTOS de las ecuaciones; los cálculos siguen
%   empleando el punto decimal propio de MATLAB.
    s = strrep(s, '.', ',');
end

function s = normaliza(x)
%NORMALIZA  Convierte a texto en minúsculas y sin espacios; '' si está vacío.
    s = '';
    if isa(x,'missing'); return; end
    if isnumeric(x)
        if isempty(x) || any(isnan(x(:))); return; end
        s = lower(strrep(num2str(x),' ',''));
        return;
    end
    s = lower(strrep(char(string(x)),' ',''));
end

function [p, R2] = ajustePoli(x, y, grado)
%AJUSTEPOLI  Ajuste polinómico por mínimos cuadrados y coeficiente R².
%   R² = 1 - SSE/SST, con SSE = Σ(y - ŷ)²  y  SST = Σ(y - mean(y))².
    x = x(:).';  y = y(:).';
    ws = warning('off','MATLAB:polyfit:RepeatedPointsOrRescale');
    p  = polyfit(x, y, grado);
    warning(ws);
    yhat = polyval(p, x);
    SSE = sum((y - yhat).^2);
    SST = sum((y - mean(y)).^2);
    R2  = 1 - SSE/SST;
end

function exportarFigura(fig, rutaSinExt, cfg)
    if exist('exportgraphics','file') == 2
        exportgraphics(fig, [rutaSinExt '.png'], 'Resolution', cfg.dpiPNG);
        exportgraphics(fig, [rutaSinExt '.pdf'], 'ContentType','vector');
        if cfg.exportarSVG
            try, exportgraphics(fig, [rutaSinExt '.svg'], 'ContentType','vector'); catch, end
        end
    else
        print(fig, [rutaSinExt '.png'], '-dpng', ['-r' num2str(cfg.dpiPNG)]);
        print(fig, [rutaSinExt '.pdf'], '-dpdf', '-painters');
        if cfg.exportarSVG
            try, print(fig, [rutaSinExt '.svg'], '-dsvg'); catch, end
        end
    end
end
```