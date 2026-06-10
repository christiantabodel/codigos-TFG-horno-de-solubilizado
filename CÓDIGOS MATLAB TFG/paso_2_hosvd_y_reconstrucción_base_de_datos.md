Paso 2: HOSVD y reconstrucción de la base de datos.

En el segundo paso del proceso de implementación del modelo de orden reducido, se aplica la HOSVD a los cuatro tensores 7×7×7 construidos en el paso anterior (operando sobre el tensor centrado respecto a su media) y evalúa la capacidad de la descomposición para comprimir y reconstruir la base de datos. Para cada salida se calculan los valores singulares de los tres desdoblamientos direccionales, su energía acumulada y el error de reconstrucción, tanto relativo en norma de Frobenius como máximo absoluto, en función del número de modos retenidos por dirección. El script genera seis figuras: valores singulares por dirección, energía acumulada, error de reconstrucción frente al número de modos, comparativa entre la superficie de respuesta del CFD y su reconstrucción con dos modos, superficies de respuesta CFD de las cuatro salidas y mapa del error absoluto de T_max. De estas salidas gráficas proceden las Figuras 48, 49, 50 y 57 de la memoria; el corte de los mapas se controla desde la configuración eligiendo la resistencia que se fija y su nivel (la vista por defecto, con Q3 en el nivel central, es la de la Figura 57, y fijando Q1 en su nivel máximo se obtiene la de la Figura 50). Los resultados numéricos se guardan en resultados_hosvd.mat.


```matlab
% Paso 2: HOSVD de las 4 salidas escalares (tensores 7x7x7): valores singulares,
% energía, error de reconstrucción y mapas de respuesta
clear; clc; close all;

% 0.CONFIGURACIÓN
nombre_mat   = 'datos_preparados.mat';
fichero_out  = 'resultados_hosvd.mat';
r_demo       = 2;     % nº de modos para las figuras de mapa
eje_fijo     = 3;     % resistencia FIJADA en los mapas (1=Q1, 2=Q2, 3=Q3)
nivel        = 4;     % nivel del eje fijado (1=min 362, 4=central 415, 7=max 468)
centrar      = true;  % HOSVD sobre el tensor centrado

% 1.CARGAR LOS TENSORES PREPARADOS
carpeta_script = fileparts(mfilename('fullpath'));
if isempty(carpeta_script); carpeta_script = pwd; end
ruta_mat = fullfile(carpeta_script, nombre_mat);
if exist(ruta_mat,'file') ~= 2
    error('No se encuentra %s. Ejecuta antes paso1_preparar_datos.m', nombre_mat);
end
S = load(ruta_mat);
tensores   = S.tensores;
col_salidas = S.col_salidas;
q1 = S.q1_niveles;  q2 = S.q2_niveles;  q3 = S.q3_niveles;
n_salidas = numel(col_salidas);
n  = numel(q1);
rangos = 1:n;
fprintf('HOSVD de %d salidas, tensores %dx%dx%d. Centrado = %d\n', n_salidas, n, n, n, centrar);

% 2.HOSVD + ENERGÍA + ERROR DE RECONSTRUCCIÓN PARA CADA SALIDA
R = struct();  SVcell = cell(n_salidas,1);  Ecell = cell(n_salidas,1);
relF = zeros(n_salidas,n);  emaxK = zeros(n_salidas,n);
for o = 1:n_salidas
    nombre = col_salidas{o};
    Tfull  = tensores.(nombre);
    mu     = centrar * mean(Tfull(:));
    Tc     = Tfull - mu;
    [U, s, core] = hosvd3(Tc);
    sv = cell(1,3);  en = cell(1,3);
    for d = 1:3
        sv{d} = s{d};  en{d} = cumsum(s{d}.^2)/sum(s{d}.^2);
    end
    SVcell{o} = sv;  Ecell{o} = en;
    for r = rangos
        cr   = core(1:r,1:r,1:r);
        Trec = mu + reconstruct3d(cr, U{1}(:,1:r), U{2}(:,1:r), U{3}(:,1:r));
        relF(o,r)  = norm(Tfull(:)-Trec(:))/norm(Tfull(:));
        emaxK(o,r) = max(abs(Tfull(:)-Trec(:)));
    end
    R(o).nombre = nombre;  R(o).mu = mu;  R(o).U = U;  R(o).s = s;  R(o).core = core;
end

% 3.RESUMEN POR CONSOLA
fprintf('\nRESUMEN PASO 2 (HOSVD)\n');
fprintf('%-18s %9s %9s %9s  %8s %8s  %10s\n', 'Salida','sigma1','sigma2','sigma3','r(<1%)','r(<0.1%)','errMax r=2');
for o = 1:n_salidas
    s1 = SVcell{o}{1};
    r1 = find(relF(o,:) < 0.01,  1, 'first');   if isempty(r1), r1 = NaN; end
    r2 = find(relF(o,:) < 0.001, 1, 'first');   if isempty(r2), r2 = NaN; end
    fprintf('%-18s %9.3g %9.3g %9.3g  %8d %8d  %8.3f K\n', col_salidas{o}, s1(1), s1(2), s1(3), r1, r2, emaxK(o,2));
end

% 4.FIGURAS
estilo  = {'-o','-s','-^'};
cols    = [0 0.447 0.741; 0.850 0.325 0.098; 0.466 0.674 0.188];
nom_dir = {'direccion Q_1','direccion Q_2','direccion Q_3'};

% Figura 1: valores singulares
figure('Name','Valores singulares','Color','w','Position',[80 80 940 680]);
for o = 1:n_salidas
    subplot(2,2,o);
    for d = 1:3
        semilogy(1:n, SVcell{o}{d}, estilo{d}, 'Color',cols(d,:), 'LineWidth',1.6, 'MarkerSize',6, 'MarkerFaceColor',cols(d,:)); hold on;
    end
    grid on; set(gca,'GridAlpha',0.3,'FontSize',11,'XTick',1:n); xlim([1 n]);
    xlabel('Indice de modo, r'); ylabel('\sigma_r'); title(col_salidas{o},'Interpreter','none');
    if o==1, legend(nom_dir,'Location','northeast','FontSize',9); end
end

% Figura 2: energía acumulada
figure('Name','Energia acumulada','Color','w','Position',[120 60 940 680]);
for o = 1:n_salidas
    subplot(2,2,o);
    for d = 1:3
        plot(1:n, Ecell{o}{d}, estilo{d}, 'Color',cols(d,:), 'LineWidth',1.6, 'MarkerSize',6, 'MarkerFaceColor',cols(d,:)); hold on;
    end
    yline(0.999,'k--','99.9%','FontSize',8,'LabelHorizontalAlignment','left','LabelVerticalAlignment','bottom');
    grid on; set(gca,'GridAlpha',0.3,'FontSize',11,'XTick',1:n); xlim([1 n]); ylim([0 1.02]);
    xlabel('Indice de modo, r'); ylabel('Suma acumulada / Total'); title(col_salidas{o},'Interpreter','none');
    if o==1, legend(nom_dir,'Location','southeast','FontSize',9); end
end

% Figura 3: error de reconstrucción vs nº de modos
figure('Name','Error de reconstruccion vs nº de modos','Color','w');
subplot(1,2,1);
semilogy(rangos, relF'*100, '-o', 'LineWidth',1.4, 'MarkerSize',5);
grid on; xlabel('N\circ de modos por direccion (r)'); ylabel('Error relativo Frobenius [%]'); title('Error relativo');
legend(col_salidas,'Interpreter','none','Location','northeast');
subplot(1,2,2);
semilogy(rangos, emaxK', '-o', 'LineWidth',1.4, 'MarkerSize',5);
grid on; xlabel('N\circ de modos por direccion (r)'); ylabel('Error maximo absoluto [K]'); title('Error maximo');

% Preparación común de los MAPAS (se fija la resistencia eje_fijo)
ejes   = {'Q_1','Q_2','Q_3'};
Qs     = {q1, q2, q3};
libres = setdiff(1:3, eje_fijo);            % las dos resistencias del mapa
xL = Qs{libres(1)}/1e3;   yL = Qs{libres(2)}/1e3;
dx = xL(2)-xL(1);  dy = yL(2)-yL(1);
xe = [xL(1)-dx/2; xL(:)+dx/2];   % bordes de celda (puntos medios)
ye = [yL(1)-dy/2; yL(:)+dy/2];
xlab = sprintf('%s [kW/m^3]', ejes{libres(1)});
ylab = sprintf('%s [kW/m^3]', ejes{libres(2)});
txtfijo = sprintf('%s = %.0f kW/m^3', ejes{eje_fijo}, Qs{eje_fijo}(nivel)/1e3);
idxf = repmat({':'},1,3);  idxf{eje_fijo} = nivel;   % subíndices del corte 2D

% Figura 4: superficie de respuesta CFD vs HOSVD
o = 1;  nombre = col_salidas{o};
Tfull = tensores.(nombre);
Trec2 = R(o).mu + reconstruct3d(R(o).core(1:r_demo,1:r_demo,1:r_demo), R(o).U{1}(:,1:r_demo), R(o).U{2}(:,1:r_demo), R(o).U{3}(:,1:r_demo));
Mcfd = squeeze(Tfull(idxf{:}))';
Mrec = squeeze(Trec2(idxf{:}))';
clim = [min(Mcfd(:)), max(Mcfd(:))];
figure('Name','Superficie de respuesta CFD vs HOSVD','Color','w');
subplot(1,2,1);
imagesc(xL, yL, Mcfd); set(gca,'YDir','normal','XTick',xL,'YTick',yL); xtickformat('%.0f'); ytickformat('%.0f'); caxis(clim); colorbar;
xlabel(xlab); ylabel(ylab); title(sprintf('%s  -  CFD', nombre),'Interpreter','none');
subplot(1,2,2);
imagesc(xL, yL, Mrec); set(gca,'YDir','normal','XTick',xL,'YTick',yL); xtickformat('%.0f'); ytickformat('%.0f'); caxis(clim); colorbar;
xlabel(xlab); ylabel(ylab); title(sprintf('Reconstruccion HOSVD (r=%d)', r_demo));

% Figura 5: superficies de respuesta de las 4 salidas (CFD)
figure('Name','Superficies de respuesta de las 4 salidas (CFD)','Color','w','Position',[80 80 950 720]);
for o = 1:n_salidas
    subplot(2,2,o);
    Tn = tensores.(col_salidas{o});
    M  = squeeze(Tn(idxf{:}))';
    imagesc(xL, yL, M); set(gca,'YDir','normal','XTick',xL,'YTick',yL); xtickformat('%.0f'); ytickformat('%.0f'); colorbar;
    xlabel(xlab); ylabel(ylab); title(col_salidas{o},'Interpreter','none');
end

% Figura 6: mapa de error |CFD - HOSVD| para T_max
o_mx = find(strcmp(col_salidas,'T_max_K'));  if isempty(o_mx), o_mx = 3; end
Tmx  = tensores.(col_salidas{o_mx});
Trec_mx = R(o_mx).mu + reconstruct3d(R(o_mx).core(1:r_demo,1:r_demo,1:r_demo), R(o_mx).U{1}(:,1:r_demo), R(o_mx).U{2}(:,1:r_demo), R(o_mx).U{3}(:,1:r_demo));
Merr = squeeze(abs(Tmx(idxf{:}) - Trec_mx(idxf{:})))';
figure('Name','Mapa de error HOSVD (T_max)','Color','w');
imagesc(xL, yL, Merr); set(gca,'YDir','normal','XTick',xL,'YTick',yL); xtickformat('%.0f'); ytickformat('%.0f');
anc = [0.001 0.000 0.014; 0.155 0.044 0.265; 0.367 0.074 0.432; 0.576 0.149 0.404; 0.796 0.281 0.299; 0.929 0.472 0.326; 0.988 0.683 0.420; 0.987 0.991 0.749];
colormap(gca, interp1(linspace(0,1,size(anc,1)), anc, linspace(0,1,256)));  % colormap tipo magma (morado->amarillo)
cb = colorbar;  cb.Label.String = '|error| [K]';
xlabel(xlab); ylabel(ylab);

% 5.GUARDAR RESULTADOS NUMÉRICOS
save(fullfile(carpeta_script, fichero_out), 'R','SVcell','Ecell','relF','emaxK','rangos','col_salidas','centrar');
fprintf('\nGuardado: %s\n', fichero_out);
fprintf('Paso 2 OK.\n');

% FUNCIONES LOCALES (HOSVD manual para tensores 3D)
function [U, s, core] = hosvd3(T)
    sz = size(T);
    T1 = reshape(T,               sz(1), sz(2)*sz(3));
    T2 = reshape(permute(T,[2 1 3]), sz(2), sz(1)*sz(3));
    T3 = reshape(permute(T,[3 1 2]), sz(3), sz(1)*sz(2));
    [U1,S1,~] = svd(T1,'econ');  [U2,S2,~] = svd(T2,'econ');  [U3,S3,~] = svd(T3,'econ');
    U = {U1, U2, U3};  s = {diag(S1), diag(S2), diag(S3)};
    core = reconstruct3d(T, U1', U2', U3');
end

function Trec = reconstruct3d(core, A, B, C)
    [r1, r2, r3] = tam3(core);
    I = size(A,1);  J = size(B,1);  K = size(C,1);
    M1 = A * reshape(core, r1, r2*r3);
    G  = reshape(M1, I, r2, r3);
    G2 = reshape(permute(G,[2 1 3]), r2, I*r3);
    M2 = B * G2;
    G  = permute(reshape(M2, J, I, r3), [2 1 3]);
    G3 = reshape(permute(G,[3 1 2]), r3, I*J);
    M3 = C * G3;
    Trec = permute(reshape(M3, K, I, J), [2 3 1]);
end

function [a,b,c] = tam3(T)
    sz = size(T);  sz(end+1:3) = 1;
    a = sz(1);  b = sz(2);  c = sz(3);
end
```