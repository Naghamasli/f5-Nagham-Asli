# Proxy image inherited from the previous team.

FROM nginx:stable-alpine

RUN rm -f /etc/nginx/conf.d/default.conf && \
    chown -R nginx:nginx /var/cache/nginx

COPY nginx.conf /etc/nginx/nginx.conf

USER nginx

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
