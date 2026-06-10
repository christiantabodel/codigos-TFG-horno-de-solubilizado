texto de ejemplo


```matlab
% Paso 3: validación predictiva por holdout estructurado: entrena con la sub-rejilla
% 4x4x4 y predice los 279 casos no vistos (HOSVD vs SVD vs interpolación 3D).

clear; clc; close all;

% 0.CONFIGURACIÓN
nombre_mat = 'datos_preparados.mat';
sub        = [1 3 5 7];     % niveles de la sub-rejilla de entrenamiento (holdout)
r_hosvd    = 3;             % modos retenidos en la HOSVD
r_svd      = 3;             % modos retenidos en la SVD plana

% 1.CARGA DE DATOS
carpeta = fileparts(mfilename('fullpath'));
if isempty(carpeta); carpeta = pwd; end
if exist(fullfile(carpeta,nombre_mat),'file') ~= 2
    error('Falta %s. Ejecuta el paso 1 antes.', nombre_mat);
end
S = load(fullfile(carpeta,nombre_mat));
tensores = S.tensores;  col_salidas = S.col_salidas;
q1 = S.q1_niveles(:);  q2 = S.q2_niveles(:);  q3 = S.q3_niveles(:);
n_sal = numel(col_salidas);

% 2.HOLDOUT ESTRUCTURADO (HOSVD vs SVD vs interpolación)
qa = q1(sub);  qb = q2(sub);  qc = q3(sub);
[I,J,K] = ndgrid(1:numel(q1), 1:numel(q2), 1:numel(q3));
in_sub  = ismember(I,sub) & ismember(J,sub) & ismember(K,sub);
test_m  = ~in_sub;
Qb_test = [q1(I(test_m)), q2(J(test_m)), q3(K(test_m))];

metodos_B = {'HOSVD','SVD','Interpolacion'};
B_true = cell(n_sal,1);  B_pred = cell(n_sal,3);  B_met = zeros(n_sal,3,4);
for o = 1:n_sal
    Tn   = tensores.(col_salidas{o});
    Tsub = Tn(sub,sub,sub);
    ytrue = Tn(test_m);
    yp_h = pred_hosvd(Tsub, qa, qb, qc, Qb_test, r_hosvd);   % HOSVD
    yp_s = pred_svd  (Tsub, qa, qb, qc, Qb_test, r_svd);     % SVD plano
    Fg   = griddedInterpolant({qa,qb,qc}, Tsub, 'spline');   % interpolación 3D
    yp_i = Fg(Qb_test(:,1), Qb_test(:,2), Qb_test(:,3));
    preds = {yp_h, yp_s, yp_i};
    for mth = 1:3
        [B_met(o,mth,1),B_met(o,mth,2),B_met(o,mth,3),B_met(o,mth,4)] = ...
            metricas(ytrue, preds{mth});
        B_pred{o,mth} = preds{mth};
    end
    B_true{o} = ytrue;
end

% 3.RESUMEN POR CONSOLA
fprintf('\nHOLDOUT ESTRUCTURADO (279 casos no vistos)\n');
for mth = 1:3
    fprintf('-- %s --\n', metodos_B{mth});
    fprintf('%-18s %10s %10s %10s\n','Salida','RMSE[K]','max[K]','R2');
    for o = 1:n_sal
        fprintf('%-18s %10.4f %10.4f %10.5f\n', col_salidas{o}, ...
            B_met(o,mth,1), B_met(o,mth,3), B_met(o,mth,4));
    end
end

% 4.FIGURAS (sin título)
col_m = lines(3);   % 1=HOSVD (azul), 2=SVD (naranja), 3=Interpolación (amarillo)

% Figura 1: comparativa de error por método (RMSE rel. | error máximo)
RMSEpct = zeros(n_sal,3);
for o = 1:n_sal
    for mth = 1:3, RMSEpct(o,mth) = 100 * B_met(o,mth,1) / mean(B_true{o}); end
end
figure('Name','Holdout: comparativa de error por metodo','Color','w','Position',[80 60 1000 460]);
subplot(1,2,1);
hb = bar(RMSEpct);
set(gca,'YScale','log','XTickLabel',col_salidas,'TickLabelInterpreter','none','FontSize',9);
ylabel('RMSE relativo [%]'); grid on; ylim([1e-4 8]);
legend(metodos_B,'Location','northwest','FontSize',8);
fac = [1.6 3.0 5.6];   % alturas escalonadas + línea guía de color desde cada barra
hold on;
for o = 1:n_sal
    gtop = max([hb(1).YEndPoints(o) hb(2).YEndPoints(o) hb(3).YEndPoints(o)]);
    for m = 1:3
        yv = RMSEpct(o,m);
        if yv>=0.1, lab=sprintf('%.3g',yv); elseif yv>=0.001, lab=sprintf('%.2g',yv); else, lab=sprintf('%.0e',yv); end
        xb = hb(m).XEndPoints(o);  yb = hb(m).YEndPoints(o);  yl = gtop*fac(m);
        plot([xb xb], [yb yl/1.12], '-', 'Color', col_m(m,:), 'LineWidth',1.0, 'HandleVisibility','off');
        text(xb, yl, lab, 'HorizontalAlignment','center','VerticalAlignment','middle','FontSize',7);
    end
end
subplot(1,2,2);
hb = bar(squeeze(B_met(:,:,3)));
set(gca,'YScale','log','XTickLabel',col_salidas,'TickLabelInterpreter','none','FontSize',9);
ylabel('Error maximo absoluto [K]'); grid on; ylim([1e-3 100]);
legend(metodos_B,'Location','northwest','FontSize',8);
fac = [1.6 3.0 5.6];
hold on;
for o = 1:n_sal
    gtop = max([hb(1).YEndPoints(o) hb(2).YEndPoints(o) hb(3).YEndPoints(o)]);
    for m = 1:3
        yv = B_met(o,m,3);
        if yv>=0.1, lab=sprintf('%.3g',yv); elseif yv>=0.001, lab=sprintf('%.2g',yv); else, lab=sprintf('%.0e',yv); end
        xb = hb(m).XEndPoints(o);  yb = hb(m).YEndPoints(o);  yl = gtop*fac(m);
        plot([xb xb], [yb yl/1.12], '-', 'Color', col_m(m,:), 'LineWidth',1.0, 'HandleVisibility','off');
        text(xb, yl, lab, 'HorizontalAlignment','center','VerticalAlignment','middle','FontSize',7);
    end
end

% Figura 2: CFD vs predicho para T_max (izq. solo HOSVD | der. los 3)
o_mx = find(strcmp(col_salidas,'T_max_K'));  if isempty(o_mx), o_mx = 3; end
yt = B_true{o_mx};  lims = [min(yt) max(yt)];
figure('Name','Holdout: CFD vs predicho T_max','Color','w','Position',[90 70 1050 500]);
subplot(1,2,1);
plot(yt, B_pred{o_mx,1}, 'o', 'MarkerSize',5, 'Color',col_m(1,:), 'MarkerFaceColor',col_m(1,:)); hold on;
plot(lims,lims,'k--','LineWidth',1.1);
grid on; axis equal tight; xlabel('CFD [K]'); ylabel('Predicho [K]');
legend('HOSVD','y = x (ideal)','Location','northwest','FontSize',8);
subplot(1,2,2);
mk = {'o','s','^'};
for mth = 1:3
    plot(yt, B_pred{o_mx,mth}, mk{mth}, 'MarkerSize',5, 'Color',col_m(mth,:)); hold on;
end
plot(lims,lims,'k--','LineWidth',1.1);
grid on; axis equal tight; xlabel('CFD [K]'); ylabel('Predicho [K]');
legend([metodos_B, {'y = x (ideal)'}],'Location','northwest','FontSize',8);

% Figura 3: distribución de errores (T_max)
figure('Name','Holdout: distribucion de errores (T_max)','Color','w','Position',[120 100 620 440]);
err = {B_pred{o_mx,1}-yt, B_pred{o_mx,2}-yt, B_pred{o_mx,3}-yt};
allerr = [err{1}; err{2}; err{3}];
edges  = linspace(min(allerr), max(allerr), 26);   % mismos bins para las tres
hS = histogram(err{2}, 'BinEdges',edges, 'FaceColor',col_m(2,:), 'FaceAlpha',0.5, 'EdgeColor','none'); hold on; % SVD
hI = histogram(err{3}, 'BinEdges',edges, 'FaceColor',col_m(3,:), 'FaceAlpha',0.5, 'EdgeColor','none');          % Interpolación
hH = histogram(err{1}, 'BinEdges',edges, 'FaceColor',col_m(1,:), 'FaceAlpha',0.5, 'EdgeColor','none');          % HOSVD
histogram(err{1}, 'BinEdges',edges, 'DisplayStyle','stairs', 'EdgeColor','k', 'LineWidth',1.8);
grid on; xlabel('Error de prediccion [K]'); ylabel('Nº de casos');
legend([hH hS hI], metodos_B, 'Location','northeast','FontSize',8);

% Figura 4 (3D): superficie T_max(Q1,Q2) a Q3 fijo, CFD vs HOSVD
nivel_q3_3d = 4;                              % nivel de Q3 fijado (1..7); 4 = central
o3d = find(strcmp(col_salidas,'T_max_K'));  if isempty(o3d), o3d = 3; end
Tn3   = tensores.(col_salidas{o3d});          % tensor completo 7x7x7 (verdad CFD)
[Q1g, Q2g] = ndgrid(q1, q2);
Mcfd3 = Tn3(:,:,nivel_q3_3d);                  % verdad CFD en el corte Q3 fijo
Tsub3 = Tn3(sub,sub,sub);                      % sub-rejilla de entrenamiento (holdout)
Qev3  = [Q1g(:), Q2g(:), repmat(q3(nivel_q3_3d), numel(Q1g), 1)];
Mpred3 = reshape(pred_hosvd(Tsub3, qa, qb, qc, Qev3, r_hosvd), size(Q1g));
figure('Name','Holdout 3D: superficie T_max(Q1,Q2) CFD vs HOSVD','Color','w','Position',[140 80 780 600]);
surf(Q1g/1e3, Q2g/1e3, Mcfd3, 'FaceAlpha',0.9, 'EdgeColor','none'); hold on;
mesh(Q1g/1e3, Q2g/1e3, Mpred3, 'FaceColor','none', 'EdgeColor','k', 'LineWidth',1.2);
colormap(parula);  cb = colorbar;  cb.Label.String = 'T_{max} [K] (CFD)';
xlabel('Q_1 [kW/m^3]'); ylabel('Q_2 [kW/m^3]'); zlabel('T_{max} [K]');
view(135,25); grid on; axis tight;
text(min(q1)/1e3, max(q2)/1e3, max(Mcfd3(:)), sprintf('  Q_3 = %.0f kW/m^3', q3(nivel_q3_3d)/1e3), ...
     'FontSize',9, 'FontWeight','bold', 'VerticalAlignment','top');
hp1 = plot3(NaN,NaN,NaN,'s','MarkerFaceColor',[.3 .5 .8],'MarkerEdgeColor','none','MarkerSize',10);
hp2 = plot3(NaN,NaN,NaN,'-k','LineWidth',1.2);
legend([hp1 hp2], {'CFD','HOSVD (prediccion)'}, 'Location','northeast','FontSize',8);

% 5.GUARDAR RESULTADOS
save(fullfile(carpeta,'resultados_prediccion.mat'), ...
     'metodos_B','B_met','B_true','B_pred','col_salidas','sub','r_hosvd','r_svd');
fprintf('Guardado: resultados_prediccion.mat\n');
fprintf('Paso 3 OK.\n');

% FUNCIONES LOCALES
function yp = pred_hosvd(Tsub, qa, qb, qc, Qev, r)
% HOSVD + interpolación de las matrices de factores (método central).
    mu = mean(Tsub(:));
    [U, ~, core] = hosvd3(Tsub - mu);
    A = U{1}(:,1:r);  B = U{2}(:,1:r);  C = U{3}(:,1:r);
    cr = core(1:r,1:r,1:r);
    N = size(Qev,1);  yp = zeros(N,1);
    for t = 1:N
        u1 = zeros(1,r); u2 = zeros(1,r); u3 = zeros(1,r);
        for m = 1:r
            u1(m) = interp1(qa, A(:,m), Qev(t,1), 'pchip');
            u2(m) = interp1(qb, B(:,m), Qev(t,2), 'pchip');
            u3(m) = interp1(qc, C(:,m), Qev(t,3), 'pchip');
        end
        val = reconstruct3d(cr, u1, u2, u3);
        yp(t) = mu + val(1);
    end
end

function yp = pred_svd(Tsub, qa, qb, qc, Qev, r)
% SVD plano (matricial): desdobla Q1 x (Q2,Q3), SVD, interpola los modos
% (1D en Q1, 2D en (Q2,Q3)) y recompone. Baseline de comparación.
    sz = size(Tsub);  mu = mean(Tsub(:));
    M = reshape(Tsub - mu, sz(1), sz(2)*sz(3));
    [U,Sg,V] = svd(M,'econ');  s = diag(Sg);
    Ur = U(:,1:r);  Vr = V(:,1:r);  sr = s(1:r);
    Fv = cell(1,r);
    for m = 1:r
        Fv{m} = griddedInterpolant({qb,qc}, reshape(Vr(:,m), sz(2), sz(3)), 'spline');
    end
    N = size(Qev,1);  yp = zeros(N,1);
    for t = 1:N
        acc = 0;
        for m = 1:r
            acc = acc + sr(m) * interp1(qa, Ur(:,m), Qev(t,1), 'pchip') * Fv{m}(Qev(t,2), Qev(t,3));
        end
        yp(t) = mu + acc;
    end
end

function [rmse, mae, mx, r2] = metricas(y, yp)
    y = y(:);  yp = yp(:);  e = yp - y;
    rmse = sqrt(mean(e.^2));  mae = mean(abs(e));  mx = max(abs(e));
    r2 = 1 - sum(e.^2) / sum((y - mean(y)).^2);
end

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