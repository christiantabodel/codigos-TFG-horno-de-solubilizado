Paso 1: preparación de los datos y tensorización.

Primer paso del proceso seguido para la implementación del modelo de orden reducido, correspondiente al preprocesado de los datos en MATLAB descrito en la metodología. El script lee el fichero con los resultados del barrido (admite tanto Excel como CSV) y somete la base de datos a una serie de comprobaciones de integridad: analiza que estén presentes las columnas de las tres entradas y de las cuatro salidas, que no existan valores faltantes ni ternas (Q1, Q2, Q3) duplicadas, que cada parámetro presente sus siete niveles equiespaciados y que la rejilla 7×7×7 esté completa, con las 343 combinaciones representadas una única vez. Superadas estas comprobaciones, asigna a cada design point su celda (i, j, k) en función de sus valores de potencia y construye un tensor de tercer orden 7×7×7 por cada salida térmica, verificando después que cada registro de la tabla coincide con su celda del tensor. El resultado se guarda en un fichero llamado datos_preparados.mat, del que parten los pasos siguientes de la cadena.


```matlab
% paso1_preparar_datos.m
% Fase 1: lee la base de datos CFD (343 design points), comprueba que la malla
% 7x7x7 esté completa y construye un tensor 7x7x7 por salida 
clear; clc; close all;

% 0.CONFIGURACIÓN 
nombre_fichero = 'tabla_resultados_343_final_v2.xlsx';

n_niveles    = 7;                                   % niveles por parámetro
col_entradas = {'Q1_Wm3','Q2_Wm3','Q3_Wm3'};        % entradas
col_salidas  = {'T_avg_belt_K','T_avg_interior_K','T_max_K','T_min_K'}; % salidas
fichero_salida = 'datos_preparados.mat';
tol_uniforme   = 1e-6;        % tolerancia relativa para el paso de la malla

% 1.LOCALIZAR Y LEER LA BASE DE DATOS
carpeta_script = fileparts(mfilename('fullpath'));
if isempty(carpeta_script)            % por si se ejecuta por secciones
    carpeta_script = pwd;
end
ruta_fichero = fullfile(carpeta_script, nombre_fichero);

if exist(ruta_fichero,'file') ~= 2
    error(['No se encuentra el fichero de datos:\n  %s\n' ...
           'Coloca el .xlsx o .csv en la misma carpeta que este script, ' ...
           'o cambia la variable "nombre_fichero".'], ruta_fichero);
end

[~,~,ext] = fileparts(ruta_fichero);
fprintf('Leyendo base de datos: %s\n', nombre_fichero);

if strcmpi(ext,'.csv')
    % CSV en formato español: separador ';' y decimal ','.
    opts = detectImportOptions(ruta_fichero, 'FileType','text', 'Delimiter',';');
    % Todas las columnas menos 'Name' son numéricas con coma decimal.
    vars_num = opts.VariableNames(~strcmpi(opts.VariableNames,'Name'));
    opts = setvartype(opts, vars_num, 'double');
    opts = setvaropts(opts, vars_num, 'DecimalSeparator', ',');
    T = readtable(ruta_fichero, opts);
else
    % Excel: lectura directa.
    T = readtable(ruta_fichero);
end

n_filas = height(T);
fprintf('  Filas leídas: %d   |   Columnas: %d\n', n_filas, width(T));

% 2.COMPROBAR COLUMNAS, NaN Y DUPLICADOS
columnas_necesarias = [col_entradas, col_salidas];
faltan = columnas_necesarias(~ismember(columnas_necesarias, T.Properties.VariableNames));
if ~isempty(faltan)
    error('Faltan columnas en la base de datos: %s', strjoin(faltan, ', '));
end
fprintf('  Entradas: %s\n', strjoin(col_entradas,', '));
fprintf('  Salidas : %s\n', strjoin(col_salidas ,', '));

% Valores faltantes en las columnas que usamos.
datos_num = T{:, columnas_necesarias};
if any(isnan(datos_num(:)))
    error('Hay %d valores faltantes (NaN) en columnas necesarias.', ...
          sum(isnan(datos_num(:))));
end
fprintf('  Valores faltantes (NaN): 0  -> OK\n');

% Combinaciones Q1-Q2-Q3 duplicadas.
Q = T{:, col_entradas};
n_dup = n_filas - size(unique(Q,'rows'),1);
if n_dup > 0
    error('Hay %d combinaciones Q1-Q2-Q3 duplicadas.', n_dup);
end
fprintf('  Combinaciones Q1-Q2-Q3 duplicadas: 0  -> OK\n');

% 3.NIVELES Y MALLA REGULAR DE Q1, Q2, Q3
niveles = cell(1,3);                  % niveles ordenados (asc.) de cada Q
for p = 1:3
    v = unique(T.(col_entradas{p}));  % unique devuelve orden ascendente
    niveles{p} = v(:);
    fprintf('  %-7s: %d niveles  [%.3f ... %.3f] W/m3\n', ...
            col_entradas{p}, numel(v), v(1), v(end));
    if numel(v) ~= n_niveles
        error('%s tiene %d niveles, se esperaban %d.', ...
              col_entradas{p}, numel(v), n_niveles);
    end
    pasos = diff(v);                  % comprobar equiespaciado
    if max(abs(pasos - pasos(1))) > tol_uniforme*abs(pasos(1))
        warning('Los niveles de %s no están perfectamente equiespaciados.', ...
                col_entradas{p});
    end
end
q1_niveles = niveles{1};
q2_niveles = niveles{2};
q3_niveles = niveles{3};

% 4.COMPROBAR MALLA 7x7x7 COMPLETA
n_esperado = n_niveles^3;
if n_filas ~= n_esperado
    error('Hay %d filas, pero una malla %dx%dx%d necesita %d.', ...
          n_filas, n_niveles, n_niveles, n_niveles, n_esperado);
end

% Índice (i,j,k) de cada fila a partir de su valor de Q (sin asumir orden).
i_idx = zeros(n_filas,1);  j_idx = zeros(n_filas,1);  k_idx = zeros(n_filas,1);
for f = 1:n_filas
    i_idx(f) = find(q1_niveles == T.(col_entradas{1})(f));
    j_idx(f) = find(q2_niveles == T.(col_entradas{2})(f));
    k_idx(f) = find(q3_niveles == T.(col_entradas{3})(f));
end

% Cada celda de la malla debe aparecer exactamente una vez.
lin = sub2ind([n_niveles n_niveles n_niveles], i_idx, j_idx, k_idx);
if numel(unique(lin)) ~= n_esperado
    error('La malla 7x7x7 no está completa: hay celdas vacías o repetidas.');
end
fprintf('\n  Malla %dx%dx%d COMPLETA: %d combinaciones únicas -> OK\n', ...
        n_niveles, n_niveles, n_niveles, n_esperado);

% 5.CONSTRUIR UN TENSOR 7x7x7 POR CADA SALIDA
% tensores.(salida)(i,j,k) = valor de esa salida en (Q1_i, Q2_j, Q3_k)
tensores  = struct();
n_salidas = numel(col_salidas);
for s = 1:n_salidas
    nombre  = col_salidas{s};
    Tn      = nan(n_niveles, n_niveles, n_niveles);   % init con NaN
    valores = T.(nombre);
    for f = 1:n_filas
        Tn(i_idx(f), j_idx(f), k_idx(f)) = valores(f);
    end
    if any(isnan(Tn(:)))                 % si quedan NaN, faltaba algún DP
        error('El tensor de %s tiene huecos (NaN). Malla incompleta.', nombre);
    end
    tensores.(nombre) = Tn;
end
fprintf('  Tensores construidos: %d (uno por salida), tamaño %dx%dx%d\n', ...
        n_salidas, n_niveles, n_niveles, n_niveles);

% 6.VERIFICACIÓN: cada DP de la tabla coincide con su celda del tensor
max_err = 0;
for s = 1:n_salidas
    Tn      = tensores.(col_salidas{s});
    valores = T.(col_salidas{s});
    for f = 1:n_filas
        max_err = max(max_err, abs(Tn(i_idx(f),j_idx(f),k_idx(f)) - valores(f)));
    end
end
fprintf('  Verificación tabla<->tensor: diferencia máxima = %.3e  -> OK\n', max_err);

% 7.GUARDAR DATOS PREPARADOS
meta = struct();
meta.fichero_origen     = nombre_fichero;
meta.n_design_points    = n_filas;
meta.n_niveles          = n_niveles;
meta.col_entradas       = col_entradas;
meta.col_salidas        = col_salidas;
meta.q1_niveles         = q1_niveles;
meta.q2_niveles         = q2_niveles;
meta.q3_niveles         = q3_niveles;
meta.Q_min              = min([q1_niveles; q2_niveles; q3_niveles]);
meta.Q_max              = max([q1_niveles; q2_niveles; q3_niveles]);
meta.convencion_indices = 'dim1=Q1, dim2=Q2, dim3=Q3 (todos ascendentes)';

ruta_salida = fullfile(carpeta_script, fichero_salida);
save(ruta_salida, 'tensores','meta','q1_niveles','q2_niveles','q3_niveles', ...
     'col_entradas','col_salidas','T');
fprintf('\n  Guardado: %s\n', ruta_salida);

fprintf('\nFase 1 OK. Siguiente: paso2_hosvd.m\n');
```